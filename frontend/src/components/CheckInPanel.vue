<template>
	<div class="flex flex-col bg-white rounded w-full py-6 px-4 border-none">
		<h2 class="text-lg font-bold text-gray-900">
			Hey, {{ employee?.data?.first_name }} 👋
		</h2>
		<div class="font-medium text-sm text-gray-500 mt-1.5" v-if="lastLog">
			Last {{ lastLogType }} was at {{ lastLogTime }}
		</div>
		<Button
			class="mt-4 mb-1 drop-shadow-sm py-5 text-base"
			id="open-checkin-modal"
			@click="handleCheckIn"
		>
			<template #prefix>
				<FeatherIcon
					:name="
						nextAction.action === 'IN'
							? 'arrow-right-circle'
							: 'arrow-left-circle'
					"
					class="w-4"
				/>
			</template>
			{{ nextAction.label }}
		</Button>
	</div>

	<ion-modal
		ref="modal"
		trigger="open-checkin-modal"
		:initial-breakpoint="1"
		:breakpoints="[0, 1]"
	>
		<div
			class="h-40 w-full flex flex-col items-center justify-center gap-5 p-4 mb-5"
		>
			<div class="flex flex-col gap-1.5 items-center justify-center">
				<div class="font-bold text-xl">
					{{ dayjs(checkinTimestamp).format("hh:mm:ss a") }}
				</div>
				<div class="font-medium text-gray-500 text-sm">
					{{ dayjs().format("D MMM, YYYY") }}
				</div>
			</div>
			<Button
				variant="solid"
				class="w-full py-5 text-sm"
				@click="submitLog(nextAction.action)"
			>
				Confirm {{ nextAction.label }}
			</Button>
		</div>
	</ion-modal>
</template>

<script setup>
import { createListResource, toast, FeatherIcon } from "frappe-ui"
import axios from "axios"
import { computed, inject, ref, onMounted, onBeforeUnmount, watch } from "vue"
import { IonModal, modalController } from "@ionic/vue"
import * as turf from "@turf/turf"
import { FrappeApp } from "frappe-js-sdk"

const getSiteName = () => {
	if (
		window.frappe?.boot?.versions?.frappe &&
		(window.frappe.boot.versions.frappe.startsWith("15") ||
			window.frappe.boot.versions.frappe.startsWith("16"))
	) {
		return window.frappe?.boot?.sitename ?? import.meta.env.VITE_SITE_NAME
	}
	return import.meta.env.VITE_SITE_NAME
}

const DOCTYPE = "Employee Checkin"
const siteurl = getSiteName()
const frappe = new FrappeApp(siteurl)
const db = frappe.db()
let centerCoordinates = null
let custom_area = null
let centerCoordinatesArray = null

const socket = inject("$socket")
const employee = inject("$employee")
const dayjs = inject("$dayjs")
const checkinTimestamp = ref(null)

const checkins = createListResource({
	doctype: DOCTYPE,
	fields: [
		"name",
		"employee",
		"employee_name",
		"log_type",
		"time",
		"device_id",
	],
	filters: {
		employee: employee.data.name,
	},
	orderBy: "time desc",
})
checkins.reload()

const lastLog = computed(() => {
	if (checkins.list.loading || !checkins.data) return {}
	return checkins.data[0]
})

const lastLogType = computed(() => {
	return lastLog?.value?.log_type === "IN" ? "check-in" : "check-out"
})

const nextAction = computed(() => {
	return lastLog?.value?.log_type === "IN"
		? { action: "OUT", label: "Check Out" }
		: { action: "IN", label: "Check In" }
})

const lastLogTime = computed(() => {
	const timestamp = lastLog?.value?.time
	const formattedTime = dayjs(timestamp).format("hh:mm a")

	if (dayjs(timestamp).isToday()) return formattedTime
	else if (dayjs(timestamp).isYesterday()) return `${formattedTime} yesterday`
	else if (dayjs(timestamp).isSame(dayjs(), "year"))
		return `${formattedTime} on ${dayjs(timestamp).format("D MMM")}`
	return `${formattedTime} on ${dayjs(timestamp).format("D MMM, YYYY")}`
})

const userPosition = ref(null)
const isInside = ref(null)
const count = ref(0)
let userdetails = null

const fetchShiftAssignments = async () => {
	try {
		const docs = await db.getDocList("Shift Assignment", {
			filters: [["employee_name", "=", employee.data.employee_name]],
		})
		if (docs && docs.length > 0) {
			const docID = docs[0].name
			const mainDoc = await db.getDoc("Shift Assignment", docID)
			userdetails = mainDoc.shift_type
		} else {
			console.log("No shift assignments found for the employee.")
		}
	} catch (error) {
		console.error(error)
	}
}

const getdocfun = async () => {
	try {
		if (userdetails) {
			const docs = await db.getDoc("Shift Type", userdetails)
			centerCoordinates = docs.custom_coordinates_
			custom_area = docs.custom_area
			centerCoordinatesArray = centerCoordinates.split(",").map(Number)
		} else {
			console.log("User details not set.")
		}
	} catch (error) {
		console.error(error)
	}
}

onMounted(async () => {
	if (employee.data.name) {
		await fetchShiftAssignments()
		await getdocfun()
	}

	if (navigator.geolocation && count.value === 0) {
		navigator.geolocation.getCurrentPosition(
			(position) => {
				const { latitude, longitude } = position.coords
				userPosition.value = [latitude, longitude]
				console.log("Retrieved location:", userPosition.value)
				count.value = 1
			},
			(error) => {
				console.error("Error getting location:", error)
				toast({
					title: "Error",
					text: "Unable to fetch your location. Please ensure location services are enabled.",
					icon: "alert-circle",
					position: "bottom-center",
					iconClasses: "text-red-500",
				})
			},
			{
				enableHighAccuracy: true,
				timeout: 5000,
				maximumAge: 0,
			}
		)
	}

	socket.emit("doctype_subscribe", DOCTYPE)
	socket.on("list_update", (data) => {
		if (data.doctype == DOCTYPE) {
			checkins.reload()
		}
	})
})

onBeforeUnmount(() => {
	socket.emit("doctype_unsubscribe", DOCTYPE)
	socket.off("list_update")
})

watch(
	() => employee.data,
	(newVal) => {
		if (newVal && newVal.name) {
			fetchShiftAssignments()
		}
	}
)

watch([userPosition, count], async () => {
	if (userPosition.value && count.value === 1) {
		const centerPoint = turf.point(centerCoordinatesArray)
		const userPoint = turf.point(userPosition.value)
		const radius = custom_area
		const options = {
			steps: 64,
			units: "kilometers",
		}
		const circle = turf.circle(centerPoint, radius, options)
		console.log("User position:", userPosition.value)
		console.log("Center coordinates2:", centerPoint)
		console.log("Circle:", circle)
		console.log("radius:", radius)
		const inside = turf.booleanPointInPolygon(userPoint, circle)
		console.log("Is inside:", inside)
		isInside.value = inside
		count.value = 2
	}
})

const handleCheckIn = () => {
	if (count.value === 1) {
		if (isInside.value) {
			checkinTimestamp.value = dayjs().format("YYYY-MM-DD HH:mm:ss")
			modalController.present("open-checkin-modal")
		} else {
			toast({
				title: "Error",
				text: "You are outside the specified area and cannot check in.",
				icon: "alert-circle",
				position: "bottom-center",
				iconClasses: "text-red-500",
			})
		}
	} else {
		toast({
			title: "Your Location",
			text: userPosition,
			icon: "alert-circle",
			position: "bottom-center",
			iconClasses: "text-red-500",
		})
	}
	checkinTimestamp.value = dayjs().format("YYYY-MM-DD HH:mm:ss")
}

const submitLog = (logType) => {
	if (!isInside.value) {
		toast({
			title: "Error",
			text: "You are outside the specified area and cannot check in.",
			icon: "alert-circle",
			position: "bottom-center",
			iconClasses: "text-red-500",
		})
		return
	}

	const action = logType === "IN" ? "Check-in" : "Check-out"

	checkins.insert.submit(
		{
			employee: employee.data.name,
			username: employee.data.name,
			log_type: logType,
			time: checkinTimestamp.value,
		},
		{
			onSuccess() {
				modalController.dismiss()
				toast({
					title: "Success",
					text: `${action} successful!`,
					icon: "check-circle",
					position: "bottom-center",
					iconClasses: "text-green-500",
				})
			},
			onError() {
				toast({
					title: "Error",
					text: `${action} failed!`,
					icon: "alert-circle",
					position: "bottom-center",
					iconClasses: "text-red-500",
				})
			},
		}
	)
}
</script>