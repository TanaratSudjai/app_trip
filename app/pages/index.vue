<script setup lang="ts">
import { onMounted, ref } from 'vue'
const config = useRuntimeConfig()
const apiBase = config.public.apiBase
const person = ref(0)
const perPerson = ref([])
const name = ref('')
const res_name = ref('')
const money = ref(0)
const name_id = ref('')
const file = ref<File | null>(null)
// state
const total = ref<number | null>(null)
const loadingTotal = ref(false)

// ฟังก์ชัน fetch total
async function fetchTotal() {
    try {
        loadingTotal.value = true
        const res = await $fetch<{ total_collected: number }>(`${apiBase}/dashboard/total`)
        total.value = Array.isArray(res) ? res[0]?.total_collected || 0 : res.total_collected || 0
    } catch (err) {
        console.error("fetch total error:", err)
        total.value = 0
    } finally {
        loadingTotal.value = false
    }
}
function handleFileChange(event: Event) {
    const target = event.target as HTMLInputElement
    if (target.files && target.files.length > 0) {
        file.value = target.files[0]  // เก็บไฟล์ตัวแรก
    } else {
        file.value = null
    }
}
// จ่ายต่อคน
async function getPerPerson() {
    const res = await fetch(`${apiBase}/dashboard/per-person`)
    const data = await res.json()
    person.value = data.length
    perPerson.value = data
    money.value = perPerson.value[0]?.total_paid
}
async function addPerson() {
    if (!name.value.trim()) {
        alert('กรุณากรอกชื่อ')
        return
    }
    try {
        const res = await fetch(`${apiBase}/friends`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ name: name.value })
        })
        if (!res.ok) {
            throw new Error('เกิดข้อผิดพลาดในการเพิ่มข้อมูล')
        }
        alert('เพิ่มเเล้ว ๆ')
        name.value = ''
        getPerPerson()
    } catch (error) {
        console.error(error)
        alert('มีชื่อเเล้ว ๆ')
    }
}
async function payPerson() {
    try {
        const formData = new FormData()
        formData.append("friend_id", String(name_id.value))
        if (file.value) formData.append("slip", file.value)

        const res = await fetch("http://localhost:5000/payments", {
            method: "POST",
            body: formData
        })

        if (!res.ok) throw new Error('เกิดข้อผิดพลาดในการเพิ่มข้อมูล')
        alert('เพิ่มเเล้ว ๆ')
        name.value = ''
        getPerPerson()
    } catch (error) {
        console.error(error)
        alert('ผิดพลาดในการจ่ายเงิน')
    }
}

onMounted(() => {
    fetchTotal()
    getPerPerson()
})
</script>

<template>
    <div class="min-h-screen bg-white">

        <div class="max-w-6xl mx-auto p-2">
            <div class="container mx-auto  grid grid-cols-1 gap-3 mb-4 mt-10">
                <span class="text-md font-normal">เพิ่มรายชื่อเอง</span>
                <form class="flex gap-2" @submit.prevent="addPerson">
                    <input v-model="name" type="ชื่อ" placeholder="ชื่อ"
                        class="border border-gray-300 w-full px-4 py-2">
                    <button type="submit" class="bg-blue-400 text-white px-4 py-2 text-xs">ยืนยัน</button>
                </form>
                <span class="text-md font-normal">จ่ายเงิน (เลือกชื่อตัวเองก่อน) แนบ สลิปเงิน <div v-if="file">
                        เลือกไฟล์: {{ file.name }}
                    </div>
                    <div v-if="name_id" class="text-gray-500">
                        เลือกเเล้ว {{ res_name }}
                    </div>
                </span>

                <form class=" gap-2 grid grid-cols-1" @submit.prevent="payPerson">
                    <input type="file" class="border border-gray-300 w-full px-4 py-2" @change="handleFileChange" />
                    <button type="submit" class="bg-blue-400 text-white px-4 py-2 text-xs text-nowrap">จ่าย</button>
                </form>
            </div>
            <!-- เพิ่มคน -->
            <div class="flex gap-2 justify-between items-center px-3 mb-5">
                <h1 class="text-3xl font-normal text-gray-800 ">ภาพรวม </h1>

            </div>

            <!-- รวมยอดทั้งหมด -->
            <div class="bg-white  rounded px-6 py-1 mb-6 border border-gray-300">
                <h2 class="text-md font-normal text-gray-700 mb-3">ยอดรวมทั้งหมด <span
                        class="font-bold text-red-500">ต้องมียอด {{ person * money * 6
                            || 0 }}</span>
                    <br>
                    <span class="text-xs">( โดยคิดจาก จำนวนคน * 100 * จำนวนเดือน คือ 6 เดือน )</span>
                </h2>
                <div v-if="loadingTotal" class="text-gray-500">กำลังโหลด...</div>
                <div v-else class="text-md font-bold text-blue-400">
                    ตอนนี้ทั้งหมดมี {{ total || 0 }} บาท
                </div>

            </div>
            <!-- จ่ายต่อคน -->
            <div class="bg-white rounded  py-1 mb-2">
                <h2 class="text-md font-normal text-gray-700 mb-6">ยอดที่แต่ละคนจ่าย</h2>
                <div class="space-y-3 grid grid-cols-2 gap-1">
                    <div v-for="p in perPerson" :key="p.nickname"
                        class="flex justify-between items-center bg-white  rounded p-2 border border-gray-300 " :class="[
                            'flex justify-between items-center bg-white rounded p-2 border border-gray-300 cursor-pointer transition',
                            name_id === p.friend_id ? 'bg-blue-100 border-blue-400' : 'hover:shadow'
                        ]" @click="name_id = p.friend_id, res_name = p.nickname">
                        <span class="font-medium text-gray-800">{{ p.nickname }}</span>
                        <span class="text-blue-400 font-bold text-sm">{{ p.total_paid || 0 }} ฿</span>
                    </div>
                </div>
            </div>



        </div>
    </div>
</template>
