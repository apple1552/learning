# learning
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>강의실 대여 시스템</title>
    <!-- Chosen Palette: Warm Neutrals (Soft Grays, Muted Blues, Subtle Accents) -->
    <!-- Application Structure Plan: 이 SPA는 강의실 대여 및 관리를 위한 대시보드 형태로 설계되었습니다. 상단 내비게이션을 통해 '강의실 예약', '강의실 정보', '내 예약 현황', '관리자 대시보드' (관리자만 접근 가능) 섹션으로 이동할 수 있습니다. '강의실 예약' 섹션은 달력 기반으로 날짜를 선택하면 해당 날짜의 강의실별 대여 가능 시간대가 표시됩니다. 사용자는 가능한 시간대를 클릭하여 예약 신청 모달을 띄울 수 있습니다. '강의실 정보' 섹션은 각 강의실의 상세 정보를 보여주며, 각 강의실별로 '예약하기' 버튼을 통해 날짜 선택 모달을 띄워 예약할 수 있습니다. '내 예약 현황' 섹션은 로그인한 사용자의 모든 예약 요청 목록과 상태를 보여줍니다. '관리자 대시보드'는 모든 대기 중인 예약 요청을 표시하고, 관리자가 이를 승인하거나 거절할 수 있는 기능을 제공합니다. 이 구조는 사용자가 자신의 예약 상태를 쉽게 확인하고, 관리자는 효율적으로 예약을 관리하며, 두 가지 예약 흐름을 명확히 구분하여 사용자 편의성을 높이도록 설계되었습니다. -->
    <!-- Visualization & Content Choices: 1. 달력: HTML/CSS 그리드와 JavaScript를 사용하여 월별 달력을 동적으로 생성합니다. 날짜 클릭 시 해당 날짜의 예약 가능 여부를 표시합니다. 2. 강의실 예약 가능 시간: 각 강의실의 시간대별 예약 상태(가능, 예약 중, 승인 대기)를 텍스트와 색상으로 표시합니다. '예약하기' 버튼을 통해 상호작용을 유도합니다. 3. 예약 모달: 사용자 입력을 위한 폼을 포함한 모달 UI를 JavaScript로 제어합니다. 4. 예약 목록: 사용자와 관리자 모두에게 예약 목록을 테이블 형태로 제공하며, 상태(승인, 거절, 대기)를 명확히 표시합니다. 관리자 화면에서는 승인/거절 버튼을 추가하여 상호작용을 구현합니다. 5. 사용자 인증 및 데이터 저장: Firebase Authentication 대신 사용자 코드(localStorage에 저장)를 사용하여 사용자를 식별하며, Firestore를 사용하여 강의실 정보 및 예약 데이터를 실시간으로 저장하고 동기화합니다. -->
    <!-- CONFIRMATION: NO SVG graphics used. NO Mermaid JS used. -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getFirestore, collection, addDoc, query, where, getDocs, doc, updateDoc, onSnapshot, orderBy } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Firebase Configuration (provided by Canvas environment)
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

        let app, db;
        let userCode = localStorage.getItem('userCode') || null; // Load user code from local storage
        let isAdmin = (userCode === "lc1001"); // Admin status based on loaded code

        // Custom Message Box
        function showMessage(message, type = 'info') {
            const messageBox = document.getElementById('message-box');
            const messageText = document.getElementById('message-text');
            messageText.textContent = message;
            messageBox.className = `fixed bottom-4 right-4 p-4 rounded-lg shadow-lg z-50 text-white`;
            if (type === 'success') {
                messageBox.classList.add('bg-green-500');
            } else if (type === 'error') {
                messageBox.classList.add('bg-red-500');
            } else {
                messageBox.classList.add('bg-blue-500');
            }
            messageBox.classList.remove('hidden');
            setTimeout(() => {
                messageBox.classList.add('hidden');
            }, 3000);
        }

        // Initialize Firebase
        function initializeFirebase() {
            try {
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                console.log("Firebase initialized.");
                renderApp(); // Initial render after Firebase is ready
            } catch (error) {
                console.error("Firebase 초기화 오류:", error);
                showMessage("Firebase 초기화에 실패했습니다. " + error.message, 'error');
            }
        }

        // Classroom Data (Hardcoded for simplicity, could be fetched from Firestore)
        const classrooms = [
            { id: 'roomA', name: '강의실 A', capacity: 30, description: '최대 30명 수용 가능한 대형 강의실. 빔 프로젝터, 화이트보드 완비.' },
            { id: 'roomB', name: '강의실 B', capacity: 15, description: '최대 15명 수용 가능한 중형 강의실. 그룹 스터디 및 소규모 회의에 적합.' },
            { id: 'roomC', name: '강의실 C', capacity: 10, description: '최대 10명 수용 가능한 소형 강의실. 개인 과외 및 면접 공간으로 활용.' }
        ];

        let currentMonth = new Date();
        let selectedDate = new Date(); // Default to today

        // For classroom-first booking modal
        let currentClassroomForBooking = null;
        let currentMonthForModal = new Date();
        let selectedDateForModal = new Date();

        const timeSlots = [
            '09:00', '10:00', '11:00', '12:00', '13:00',
            '14:00', '15:00', '16:00', '17:00', '18:00'
        ];

        // Firestore Paths
        const getBookingsCollection = () => collection(db, `artifacts/${appId}/public/data/bookings`);

        // Render Calendar (Main Booking Section)
        function renderCalendar() {
            const calendarGrid = document.getElementById('calendar-grid');
            calendarGrid.innerHTML = '';
            const monthYearDisplay = document.getElementById('month-year-display');

            const year = currentMonth.getFullYear();
            const month = currentMonth.getMonth(); // 0-indexed

            monthYearDisplay.textContent = `${year}년 ${month + 1}월`;

            const firstDayOfMonth = new Date(year, month, 1).getDay(); // 0 (Sun) - 6 (Sat)
            const daysInMonth = new Date(year, month + 1, 0).getDate();

            // Fill leading empty days
            for (let i = 0; i < firstDayOfMonth; i++) {
                const emptyDay = document.createElement('div');
                emptyDay.className = 'p-2 text-center text-gray-400';
                calendarGrid.appendChild(emptyDay);
            }

            // Fill days of the month
            for (let day = 1; day <= daysInMonth; day++) {
                const date = new Date(year, month, day);
                const dayElement = document.createElement('div');
                dayElement.className = `p-2 text-center rounded-md cursor-pointer transition-colors duration-200`;
                dayElement.textContent = day;

                if (date.toDateString() === new Date().toDateString()) {
                    dayElement.classList.add('bg-blue-200', 'font-bold');
                }
                if (date.toDateString() === selectedDate.toDateString()) {
                    dayElement.classList.add('bg-blue-500', 'text-white', 'font-bold');
                }

                dayElement.addEventListener('click', () => {
                    selectedDate = date;
                    renderCalendar(); // Re-render to highlight selected date
                    fetchAndDisplayAvailability(date);
                });
                calendarGrid.appendChild(dayElement);
            }
            fetchAndDisplayAvailability(selectedDate); // Display availability for the initially selected date
        }

        // Render Calendar (Inside Classroom-First Booking Modal)
        function renderModalCalendar() {
            const calendarGrid = document.getElementById('modal-calendar-grid');
            calendarGrid.innerHTML = '';
            const monthYearDisplay = document.getElementById('modal-month-year-display');

            const year = currentMonthForModal.getFullYear();
            const month = currentMonthForModal.getMonth(); // 0-indexed

            monthYearDisplay.textContent = `${year}년 ${month + 1}월`;

            const firstDayOfMonth = new Date(year, month, 1).getDay(); // 0 (Sun) - 6 (Sat)
            const daysInMonth = new Date(year, month + 1, 0).getDate();

            for (let i = 0; i < firstDayOfMonth; i++) {
                const emptyDay = document.createElement('div');
                emptyDay.className = 'p-2 text-center text-gray-400';
                calendarGrid.appendChild(emptyDay);
            }

            for (let day = 1; day <= daysInMonth; day++) {
                const date = new Date(year, month, day);
                const dayElement = document.createElement('div');
                dayElement.className = `p-2 text-center rounded-md cursor-pointer transition-colors duration-200`;
                dayElement.textContent = day;

                if (date.toDateString() === new Date().toDateString()) {
                    dayElement.classList.add('bg-blue-200', 'font-bold');
                }
                if (date.toDateString() === selectedDateForModal.toDateString()) {
                    dayElement.classList.add('bg-blue-500', 'text-white', 'font-bold');
                }

                dayElement.addEventListener('click', () => {
                    selectedDateForModal = date;
                    renderModalCalendar(); // Re-render to highlight selected date
                    fetchAndDisplayTimeSlotsForSingleClassroom(currentClassroomForBooking, date);
                });
                calendarGrid.appendChild(dayElement);
            }
            fetchAndDisplayTimeSlotsForSingleClassroom(currentClassroomForBooking, selectedDateForModal);
        }

        // Fetch and Display Classroom Availability (Main Booking Section)
        async function fetchAndDisplayAvailability(date) {
            const availabilityContainer = document.getElementById('classroom-availability');
            availabilityContainer.innerHTML = '<p class="text-center text-gray-500">불러오는 중...</p>';

            const formattedDate = date.toISOString().split('T')[0]; // YYYY-MM-DD
            const bookingsRef = getBookingsCollection();
            const q = query(bookingsRef, where('date', '==', formattedDate));

            try {
                const querySnapshot = await getDocs(q);
                const bookedSlots = {}; // { classroomId: { startTime: status } }

                querySnapshot.forEach(doc => {
                    const booking = doc.data();
                    if (!bookedSlots[booking.classroomId]) {
                        bookedSlots[booking.classroomId] = {};
                    }
                    bookedSlots[booking.classroomId][booking.startTime] = booking.status;
                });

                availabilityContainer.innerHTML = `
                    <h3 class="text-xl font-bold mb-4 text-gray-800">${formattedDate} 강의실 대여 현황</h3>
                    <div class="space-y-6"></div>
                `;
                const classroomListDiv = availabilityContainer.querySelector('div.space-y-6');

                classrooms.forEach(classroom => {
                    const classroomDiv = document.createElement('div');
                    classroomDiv.className = 'bg-white p-6 rounded-lg shadow-md';
                    classroomDiv.innerHTML = `
                        <h4 class="text-lg font-semibold mb-3 text-blue-700">${classroom.name} (${classroom.capacity}인)</h4>
                        <p class="text-gray-600 mb-4">${classroom.description}</p>
                        <div class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 gap-3"></div>
                    `;
                    const timeSlotGrid = classroomDiv.querySelector('div.grid');

                    timeSlots.forEach(slot => {
                        const status = bookedSlots[classroom.id] && bookedSlots[classroom.id][slot]
                            ? bookedSlots[classroom.id][slot]
                            : 'available';
                        
                        const slotButton = document.createElement('button');
                        slotButton.className = `w-full py-2 px-3 rounded-md text-sm font-medium transition-colors duration-200`;
                        slotButton.textContent = slot;

                        if (status === 'available') {
                            slotButton.classList.add('bg-green-100', 'text-green-800', 'hover:bg-green-200');
                            slotButton.addEventListener('click', () => openBookingPurposeModal(classroom, formattedDate, slot));
                        } else if (status === 'pending') {
                            slotButton.classList.add('bg-yellow-100', 'text-yellow-800', 'cursor-not-allowed');
                            slotButton.textContent += ' (대기)';
                            slotButton.disabled = true;
                        } else if (status === 'approved') {
                            slotButton.classList.add('bg-red-100', 'text-red-800', 'cursor-not-allowed');
                            slotButton.textContent += ' (예약 완료)';
                            slotButton.disabled = true;
                        } else if (status === 'rejected') {
                             slotButton.classList.add('bg-gray-100', 'text-gray-600', 'cursor-not-allowed');
                             slotButton.textContent += ' (거절됨)';
                             slotButton.disabled = true;
                        }
                        timeSlotGrid.appendChild(slotButton);
                    });
                    classroomListDiv.appendChild(classroomDiv);
                });

            } catch (error) {
                console.error("강의실 예약 현황 불러오기 오류:", error);
                showMessage("예약 현황을 불러오는 데 실패했습니다.", 'error');
                availabilityContainer.innerHTML = '<p class="text-center text-red-500">예약 현황을 불러올 수 없습니다.</p>';
            }
        }

        // Fetch and Display Time Slots for a Single Classroom (Inside Modal)
        async function fetchAndDisplayTimeSlotsForSingleClassroom(classroom, date) {
            const timeSlotDisplay = document.getElementById('modal-time-slots');
            timeSlotDisplay.innerHTML = '<p class="text-center text-gray-500">시간대 불러오는 중...</p>';

            const formattedDate = date.toISOString().split('T')[0]; // YYYY-MM-DD
            const bookingsRef = getBookingsCollection();
            const q = query(bookingsRef, where('date', '==', formattedDate), where('classroomId', '==', classroom.id));

            try {
                const querySnapshot = await getDocs(q);
                const bookedSlots = {};
                querySnapshot.forEach(doc => {
                    const booking = doc.data();
                    bookedSlots[booking.startTime] = booking.status;
                });

                timeSlotDisplay.innerHTML = `
                    <h4 class="text-lg font-semibold mb-3 text-gray-800">${formattedDate} - ${classroom.name} 시간대</h4>
                    <div class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-3"></div>
                `;
                const timeSlotGrid = timeSlotDisplay.querySelector('div.grid');

                timeSlots.forEach(slot => {
                    const status = bookedSlots[slot] ? bookedSlots[slot] : 'available';
                    
                    const slotButton = document.createElement('button');
                    slotButton.className = `w-full py-2 px-3 rounded-md text-sm font-medium transition-colors duration-200`;
                    slotButton.textContent = slot;

                    if (status === 'available') {
                        slotButton.classList.add('bg-green-100', 'text-green-800', 'hover:bg-green-200');
                        slotButton.addEventListener('click', () => {
                            closeClassroomSelectDateModal();
                            openBookingPurposeModal(classroom, formattedDate, slot);
                        });
                    } else if (status === 'pending') {
                        slotButton.classList.add('bg-yellow-100', 'text-yellow-800', 'cursor-not-allowed');
                        slotButton.textContent += ' (대기)';
                        slotButton.disabled = true;
                    } else if (status === 'approved') {
                        slotButton.classList.add('bg-red-100', 'text-red-800', 'cursor-not-allowed');
                        slotButton.textContent += ' (예약 완료)';
                        slotButton.disabled = true;
                    } else if (status === 'rejected') {
                        slotButton.classList.add('bg-gray-100', 'text-gray-600', 'cursor-not-allowed');
                        slotButton.textContent += ' (거절됨)';
                        slotButton.disabled = true;
                    }
                    timeSlotGrid.appendChild(slotButton);
                });

            } catch (error) {
                console.error("단일 강의실 시간대 불러오기 오류:", error);
                showMessage("시간대를 불러오는 데 실패했습니다.", 'error');
                timeSlotDisplay.innerHTML = '<p class="text-center text-red-500">시간대를 불러올 수 없습니다.</p>';
            }
        }

        // Booking Purpose Modal (Final step for booking)
        const bookingPurposeModal = document.getElementById('booking-purpose-modal');
        const bookingPurposeForm = document.getElementById('booking-purpose-form');
        let currentBookingDetails = {};

        function openBookingPurposeModal(classroom, date, slot) {
            currentBookingDetails = { classroom, date, slot };
            document.getElementById('purpose-modal-classroom-name').textContent = classroom.name;
            document.getElementById('purpose-modal-date').textContent = date;
            document.getElementById('purpose-modal-time').textContent = slot;
            document.getElementById('booking-purpose-input').value = ''; // Clear previous input
            document.getElementById('booking-user-code-input').value = userCode || ''; // Pre-fill if userCode exists
            bookingPurposeModal.classList.remove('hidden');
        }

        function closeBookingPurposeModal() {
            bookingPurposeModal.classList.add('hidden');
        }

        bookingPurposeForm.addEventListener('submit', async (e) => {
            e.preventDefault();
            const purpose = document.getElementById('booking-purpose-input').value.trim();
            const enteredUserCode = document.getElementById('booking-user-code-input').value.trim();

            if (!enteredUserCode) {
                showMessage("예약을 위해 '내 코드'를 입력해주세요.", 'error');
                return;
            }
            if (!purpose) {
                showMessage("대여 목적을 입력해주세요.", 'error');
                return;
            }

            // Update userCode globally and in localStorage when booking
            userCode = enteredUserCode;
            localStorage.setItem('userCode', userCode);
            isAdmin = (userCode === "lc1001"); // Update admin status

            const { classroom, date, slot } = currentBookingDetails;
            const bookingsRef = getBookingsCollection();

            try {
                // Check for existing pending/approved booking for the same slot
                const q = query(bookingsRef, 
                    where('classroomId', '==', classroom.id),
                    where('date', '==', date),
                    where('startTime', '==', slot),
                    where('status', 'in', ['pending', 'approved'])
                );
                const existingBookings = await getDocs(q);

                if (!existingBookings.empty) {
                    showMessage("해당 시간대는 이미 예약되었거나 예약 대기 중입니다.", 'error');
                    closeBookingPurposeModal();
                    fetchAndDisplayAvailability(new Date(date)); // Refresh main availability
                    fetchAndDisplayTimeSlotsForSingleClassroom(classroom, new Date(date)); // Refresh modal availability if open
                    return;
                }

                await addDoc(bookingsRef, {
                    classroomId: classroom.id,
                    className: classroom.name,
                    date: date,
                    startTime: slot,
                    endTime: timeSlots[timeSlots.indexOf(slot) + 1] || '다음 시간', // Simple end time for display
                    userId: userCode, // Use userCode here
                    userName: userCode, // Use userCode as userName
                    purpose: purpose,
                    status: 'pending',
                    createdAt: new Date()
                });
                showMessage("예약 신청이 완료되었습니다. 관리자 승인 후 반영됩니다.", 'success');
                closeBookingPurposeModal();
                fetchAndDisplayAvailability(new Date(date)); // Refresh main availability
                renderMyBookings(); // Refresh my bookings
                renderApp(); // Re-render app to update admin link if code was admin
            } catch (error) {
                console.error("예약 신청 오류:", error);
                showMessage("예약 신청 중 오류가 발생했습니다.", 'error');
            }
        });

        // Classroom Select Date Modal (Classroom-first booking)
        const classroomSelectDateModal = document.getElementById('classroom-select-date-modal');
        function openClassroomSelectDateModal(classroom) {
            currentClassroomForBooking = classroom;
            document.getElementById('classroom-modal-name').textContent = classroom.name;
            document.getElementById('classroom-modal-description').textContent = classroom.description;
            
            // Reset modal calendar to current month and selected date to today
            currentMonthForModal = new Date();
            selectedDateForModal = new Date();

            renderModalCalendar();
            classroomSelectDateModal.classList.remove('hidden');
        }

        function closeClassroomSelectDateModal() {
            classroomSelectDateModal.classList.add('hidden');
        }

        // My Bookings
        async function renderMyBookings() {
            const myBookingsList = document.getElementById('my-bookings-list');
            myBookingsList.innerHTML = '<p class="text-center text-gray-500">불러오는 중...</p>';

            if (!userCode) {
                myBookingsList.innerHTML = `
                    <div class="bg-white p-6 rounded-lg shadow-md text-center">
                        <p class="mb-4 text-gray-700">내 예약 내역을 확인하려면 코드를 입력해주세요.</p>
                        <input type="text" id="my-bookings-user-code-input" placeholder="내 코드 입력" class="border rounded px-3 py-2 w-full max-w-xs mb-3 text-gray-700">
                        <button id="set-my-bookings-code-btn" class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md">코드 입력</button>
                    </div>
                `;
                document.getElementById('set-my-bookings-code-btn').addEventListener('click', () => {
                    const enteredCode = document.getElementById('my-bookings-user-code-input').value.trim();
                    if (enteredCode) {
                        userCode = enteredCode;
                        localStorage.setItem('userCode', userCode);
                        isAdmin = (userCode === "lc1001"); // Update admin status
                        renderMyBookings(); // Re-render this section with bookings
                        renderApp(); // Re-render app to update admin link
                    } else {
                        showMessage("코드를 입력해주세요.", 'error');
                    }
                });
                return;
            }

            const bookingsRef = getBookingsCollection();
            const q = query(bookingsRef, where('userId', '==', userCode), orderBy('createdAt', 'desc'));

            try {
                const querySnapshot = await getDocs(q);
                if (querySnapshot.empty) {
                    myBookingsList.innerHTML = '<p class="text-center text-gray-500">아직 예약 내역이 없습니다.</p>';
                    return;
                }

                let bookingsHtml = `
                    <div class="overflow-x-auto bg-white rounded-lg shadow-md p-4">
                        <table class="min-w-full divide-y divide-gray-200">
                            <thead class="bg-gray-50">
                                <tr>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">강의실</th>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">날짜</th>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">시간</th>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">목적</th>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">상태</th>
                                </tr>
                            </thead>
                            <tbody class="bg-white divide-y divide-gray-200">
                `;
                querySnapshot.forEach(doc => {
                    const booking = doc.data();
                    let statusClass = '';
                    let statusText = '';
                    if (booking.status === 'pending') {
                        statusClass = 'bg-yellow-100 text-yellow-800';
                        statusText = '대기 중';
                    } else if (booking.status === 'approved') {
                        statusClass = 'bg-green-100 text-green-800';
                        statusText = '승인됨';
                    } else if (booking.status === 'rejected') {
                        statusClass = 'bg-red-100 text-red-800';
                        statusText = '거절됨';
                    }

                    bookingsHtml += `
                        <tr>
                            <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">${booking.className}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">${booking.date}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">${booking.startTime} - ${booking.endTime}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">${booking.purpose}</td>
                            <td class="px-6 py-4 whitespace-nowrap">
                                <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${statusClass}">${statusText}</span>
                            </td>
                        </tr>
                    `;
                });
                bookingsHtml += `</tbody></table></div>`;
                myBookingsList.innerHTML = bookingsHtml;
            } catch (error) {
                console.error("내 예약 불러오기 오류:", error);
                showMessage("내 예약 내역을 불러오는 데 실패했습니다.", 'error');
                myBookingsList.innerHTML = '<p class="text-center text-red-500">예약 내역을 불러올 수 없습니다.</p>';
            }
        }

        // Admin Dashboard
        async function renderAdminDashboard() {
            const adminDashboardList = document.getElementById('pending-bookings-list');
            adminDashboardList.innerHTML = '<p class="text-center text-gray-500">불러오는 중...</p>';

            if (!isAdmin) {
                adminDashboardList.innerHTML = '<p class="text-center text-red-500">관리자만 접근할 수 있습니다.</p>';
                return;
            }

            const bookingsRef = getBookingsCollection();
            const q = query(bookingsRef, where('status', '==', 'pending'), orderBy('createdAt', 'asc'));

            try {
                const querySnapshot = await getDocs(q);
                if (querySnapshot.empty) {
                    adminDashboardList.innerHTML = '<p class="text-center text-gray-500">대기 중인 예약 요청이 없습니다.</p>';
                    return;
                }

                let bookingsHtml = `
                    <div class="overflow-x-auto bg-white rounded-lg shadow-md p-4">
                        <table class="min-w-full divide-y divide-gray-200">
                            <thead class="bg-gray-50">
                                <tr>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">강의실</th>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">날짜</th>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">시간</th>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">신청자</th>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">목적</th>
                                    <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">관리</th>
                                </tr>
                            </thead>
                            <tbody class="bg-white divide-y divide-gray-200">
                `;
                querySnapshot.forEach(doc => {
                    const booking = doc.data();
                    bookingsHtml += `
                        <tr>
                            <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">${booking.className}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">${booking.date}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">${booking.startTime} - ${booking.endTime}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">${booking.userName} <span class="text-gray-400">(${booking.userId})</span></td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">${booking.purpose}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                                <button data-id="${doc.id}" data-status="approved" class="approve-btn bg-green-500 hover:bg-green-600 text-white px-3 py-1 rounded-md mr-2">승인</button>
                                <button data-id="${doc.id}" data-status="rejected" class="reject-btn bg-red-500 hover:bg-red-600 text-white px-3 py-1 rounded-md">거절</button>
                            </td>
                        </tr>
                    `;
                });
                bookingsHtml += `</tbody></table></div>`;
                adminDashboardList.innerHTML = bookingsHtml;

                adminDashboardList.querySelectorAll('.approve-btn').forEach(button => {
                    button.addEventListener('click', (e) => updateBookingStatus(e.target.dataset.id, 'approved'));
                });
                adminDashboardList.querySelectorAll('.reject-btn').forEach(button => {
                    button.addEventListener('click', (e) => updateBookingStatus(e.target.dataset.id, 'rejected'));
                });

            } catch (error) {
                console.error("관리자 대시보드 불러오기 오류:", error);
                showMessage("관리자 대시보드를 불러오는 데 실패했습니다.", 'error');
                adminDashboardList.innerHTML = '<p class="text-center text-red-500">관리자 대시보드를 불러올 수 없습니다.</p>';
            }
        }

        async function updateBookingStatus(bookingId, status) {
            const bookingDocRef = doc(db, `artifacts/${appId}/public/data/bookings`, bookingId);
            try {
                await updateDoc(bookingDocRef, { status: status });
                showMessage(`예약이 ${status === 'approved' ? '승인' : '거절'}되었습니다.`, 'success');
                // Re-render admin dashboard and classroom availability
                renderAdminDashboard();
                fetchAndDisplayAvailability(selectedDate);
                renderMyBookings(); // Also refresh user's own bookings
            } catch (error) {
                console.error("예약 상태 업데이트 오류:", error);
                showMessage("예약 상태 업데이트에 실패했습니다.", 'error');
            }
        }

        // Render Classroom Info Section
        function renderClassroomInfo() {
            const classroomInfoContainer = document.getElementById('classroom-info-list');
            classroomInfoContainer.innerHTML = ''; // Clear previous content

            classrooms.forEach(room => {
                const roomDiv = document.createElement('div');
                roomDiv.className = 'bg-white p-6 rounded-lg shadow-md';
                roomDiv.innerHTML = `
                    <h3 class="text-xl font-bold mb-2 text-blue-700">${room.name}</h3>
                    <p class="text-gray-600 mb-2">수용 인원: ${room.capacity}명</p>
                    <p class="text-gray-700 mb-4">${room.description}</p>
                    <button data-room-id="${room.id}" class="book-classroom-btn bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md">이 강의실 예약하기</button>
                `;
                classroomInfoContainer.appendChild(roomDiv);
            });

            classroomInfoContainer.querySelectorAll('.book-classroom-btn').forEach(button => {
                button.addEventListener('click', (e) => {
                    const roomId = e.target.dataset.roomId;
                    const selectedClassroom = classrooms.find(room => room.id === roomId);
                    if (selectedClassroom) {
                        openClassroomSelectDateModal(selectedClassroom);
                    }
                });
            });
        }

        // Navigation Logic
        function showSection(sectionId) {
            document.querySelectorAll('main section').forEach(section => {
                section.classList.add('hidden');
            });
            document.getElementById(sectionId).classList.remove('hidden');

            document.querySelectorAll('nav a').forEach(link => {
                link.classList.remove('text-blue-700', 'font-bold', 'border-b-2', 'border-blue-700');
                link.classList.add('text-gray-600', 'hover:text-blue-700');
            });
            const activeLink = document.querySelector(`nav a[href="#${sectionId}"]`);
            if (activeLink) {
                activeLink.classList.add('text-blue-700', 'font-bold', 'border-b-2', 'border-blue-700');
                activeLink.classList.remove('text-gray-600', 'hover:text-blue-700');
            }

            // Render content based on section
            if (sectionId === 'booking-section') {
                renderCalendar();
            } else if (sectionId === 'my-bookings-section') {
                renderMyBookings();
            } else if (sectionId === 'admin-section') {
                renderAdminDashboard();
            } else if (sectionId === 'classroom-info-section') {
                renderClassroomInfo();
            }
        }

        // Initial render based on user code (for admin link visibility)
        function renderApp() {
            const adminNavLink = document.getElementById('admin-nav-link');
            if (isAdmin) {
                adminNavLink.classList.remove('hidden');
            } else {
                adminNavLink.classList.add('hidden');
            }
            showSection('booking-section'); // Default view
        }

        // Event Listeners for Navigation
        document.querySelectorAll('nav a').forEach(link => {
            link.addEventListener('click', (e) => {
                e.preventDefault();
                const sectionId = e.target.getAttribute('href').substring(1);
                showSection(sectionId);
            });
        });

        // Calendar Navigation (Main)
        document.getElementById('prev-month-btn').addEventListener('click', () => {
            currentMonth.setMonth(currentMonth.getMonth() - 1);
            renderCalendar();
        });

        document.getElementById('next-month-btn').addEventListener('click', () => {
            currentMonth.setMonth(currentMonth.getMonth() + 1);
            renderCalendar();
        });

        // Calendar Navigation (Modal)
        document.getElementById('modal-prev-month-btn').addEventListener('click', () => {
            currentMonthForModal.setMonth(currentMonthForModal.getMonth() - 1);
            renderModalCalendar();
        });

        document.getElementById('modal-next-month-btn').addEventListener('click', () => {
            currentMonthForModal.setMonth(currentMonthForModal.getMonth() + 1);
            renderModalCalendar();
        });

        // Close booking purpose modal
        document.getElementById('close-purpose-modal-btn').addEventListener('click', closeBookingPurposeModal);

        // Close classroom select date modal
        document.getElementById('close-classroom-select-date-modal-btn').addEventListener('click', closeClassroomSelectDateModal);

        // Initial Firebase setup
        initializeFirebase();
    </script>

    <style>
        body {
            font-family: 'Noto Sans KR', sans-serif;
            background-color: #f8f9fa; /* Light Gray */
            color: #333;
        }
        .nav-link {
            transition: color 0.3s, border-bottom-color 0.3s;
        }
        .nav-link.active {
            color: #1e40af; /* A shade of blue */
            border-bottom-color: #1e40af;
        }
        .hidden {
            display: none;
        }
        .modal-overlay {
            background-color: rgba(0, 0, 0, 0.5);
        }
        .calendar-day {
            min-height: 80px; /* Ensure enough space for content */
        }
        /* Custom styles for message box */
        #message-box {
            animation: fadeInOut 3s forwards;
        }
        @keyframes fadeInOut {
            0% { opacity: 0; transform: translateY(20px); }
            10% { opacity: 1; transform: translateY(0); }
            90% { opacity: 1; transform: translateY(0); }
            100% { opacity: 0; transform: translateY(20px); }
        }
    </style>
</head>
<body class="min-h-screen flex flex-col">

    <!-- Loading Overlay (Removed, as authentication is no longer required at start) -->
    <div id="loading-overlay" class="fixed inset-0 bg-gray-200 bg-opacity-75 flex items-center justify-center z-[9999] hidden">
        <div class="animate-spin rounded-full h-16 w-16 border-t-4 border-b-4 border-blue-500"></div>
        <p class="ml-4 text-lg text-gray-700">로딩 중...</p>
    </div>

    <!-- Navigation Header -->
    <header class="bg-white shadow-md py-4 sticky top-0 z-50">
        <nav class="container mx-auto px-4 flex justify-between items-center">
            <h1 class="text-2xl font-bold text-blue-800">강의실 대여 시스템</h1>
            <div class="flex items-center space-x-6">
                <!-- User code input removed from header as per request -->
                <a href="#booking-section" class="nav-link text-gray-600 hover:text-blue-700 pb-1 border-b-2 border-transparent">강의실 예약</a>
                <a href="#classroom-info-section" class="nav-link text-gray-600 hover:text-blue-700 pb-1 border-b-2 border-transparent">강의실 정보</a>
                <a href="#my-bookings-section" class="nav-link text-gray-600 hover:text-blue-700 pb-1 border-b-2 border-transparent">내 예약 현황</a>
                <a id="admin-nav-link" href="#admin-section" class="nav-link text-gray-600 hover:text-blue-700 pb-1 border-b-2 border-transparent hidden">관리자 대시보드</a>
            </div>
        </nav>
    </header>

    <!-- Main Content -->
    <main class="flex-grow container mx-auto p-6 bg-white rounded-lg shadow-xl my-8">
        <!-- Classroom Booking Section (Calendar-first) -->
        <section id="booking-section" class="hidden">
            <h2 class="text-3xl font-bold mb-6 text-gray-900">강의실 예약 (날짜 선택 후)</h2>
            <p class="mb-8 text-gray-600">달력에서 날짜를 선택하여 해당 날짜의 강의실별 대여 가능 시간대를 확인하고 예약 신청을 할 수 있습니다.</p>

            <div class="bg-gray-50 p-6 rounded-lg shadow-inner mb-8">
                <div class="flex justify-between items-center mb-4">
                    <button id="prev-month-btn" class="p-2 rounded-full bg-blue-100 hover:bg-blue-200 text-blue-700 font-bold">‹</button>
                    <h3 id="month-year-display" class="text-xl font-bold text-gray-800"></h3>
                    <button id="next-month-btn" class="p-2 rounded-full bg-blue-100 hover:bg-blue-200 text-blue-700 font-bold">›</button>
                </div>
                <div class="grid grid-cols-7 gap-2 text-center font-semibold text-gray-700 mb-2">
                    <div>일</div><div>월</div><div>화</div><div>수</div><div>목</div><div>금</div><div>토</div>
                </div>
                <div id="calendar-grid" class="grid grid-cols-7 gap-2">
                    <!-- Calendar days will be rendered here by JS -->
                </div>
            </div>

            <div id="classroom-availability" class="mt-8 p-6 bg-gray-50 rounded-lg shadow-inner">
                <!-- Availability will be displayed here by JS -->
            </div>
        </section>

        <!-- Classroom Info Section (Classroom descriptions and classroom-first booking trigger) -->
        <section id="classroom-info-section" class="hidden">
            <h2 class="text-3xl font-bold mb-6 text-gray-900">강의실 정보 및 예약 (강의실 선택 후)</h2>
            <p class="mb-8 text-gray-600">각 강의실의 상세 정보를 확인하고, 원하는 강의실을 선택하여 특정 날짜에 예약할 수 있습니다.</p>
            <div id="classroom-info-list" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                <!-- Classroom info cards will be rendered here by JS -->
            </div>
        </section>

        <!-- My Bookings Section -->
        <section id="my-bookings-section" class="hidden">
            <h2 class="text-3xl font-bold mb-6 text-gray-900">내 예약 현황</h2>
            <p class="mb-8 text-gray-600">현재 입력된 코드로 신청한 강의실 예약 내역을 확인하고, 각 예약의 승인 상태를 확인할 수 있습니다.</p>
            <div id="my-bookings-list">
                <!-- My bookings will be rendered here by JS -->
            </div>
        </section>

        <!-- Admin Dashboard Section -->
        <section id="admin-section" class="hidden">
            <h2 class="text-3xl font-bold mb-6 text-gray-900">관리자 대시보드</h2>
            <p class="mb-8 text-gray-600">모든 사용자의 대기 중인 강의실 예약 요청을 확인하고, 승인 또는 거절할 수 있습니다.</p>
            <div id="pending-bookings-list">
                <!-- Pending bookings will be rendered here by JS -->
            </div>
        </section>
    </main>

    <!-- Booking Purpose Modal (Final step for booking, used by both methods) -->
    <div id="booking-purpose-modal" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 hidden modal-overlay">
        <div class="bg-white p-8 rounded-lg shadow-xl w-full max-w-md mx-4">
            <div class="flex justify-between items-center mb-6">
                <h3 class="text-2xl font-bold text-gray-900">강의실 예약 신청</h3>
                <button id="close-purpose-modal-btn" class="text-gray-500 hover:text-gray-700 text-2xl font-semibold">
                    &times;
                </button>
            </div>
            <form id="booking-purpose-form">
                <div class="mb-4">
                    <label class="block text-gray-700 text-sm font-bold mb-2">강의실:</label>
                    <p id="purpose-modal-classroom-name" class="text-lg font-semibold text-blue-600"></p>
                </div>
                <div class="mb-4">
                    <label class="block text-gray-700 text-sm font-bold mb-2">날짜:</label>
                    <p id="purpose-modal-date" class="text-lg font-semibold text-gray-800"></p>
                </div>
                <div class="mb-4">
                    <label class="block text-gray-700 text-sm font-bold mb-2">시간:</label>
                    <p id="purpose-modal-time" class="text-lg font-semibold text-gray-800"></p>
                </div>
                <div class="mb-6">
                    <label for="booking-user-code-input" class="block text-gray-700 text-sm font-bold mb-2">내 코드 입력:</label>
                    <input type="text" id="booking-user-code-input" placeholder="예약자 코드를 입력하세요 (필수)" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" required>
                </div>
                <div class="mb-6">
                    <label for="booking-purpose-input" class="block text-gray-700 text-sm font-bold mb-2">대여 목적:</label>
                    <textarea id="booking-purpose-input" rows="3" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline resize-none" placeholder="대여 목적을 입력하세요." required></textarea>
                </div>
                <div class="flex justify-end">
                    <button type="submit" class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-md focus:outline-none focus:shadow-outline">신청하기</button>
                </div>
            </form>
        </div>
    </div>

    <!-- Classroom Select Date Modal (For classroom-first booking) -->
    <div id="classroom-select-date-modal" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 hidden modal-overlay">
        <div class="bg-white p-8 rounded-lg shadow-xl w-full max-w-2xl mx-4">
            <div class="flex justify-between items-center mb-6">
                <h3 class="text-2xl font-bold text-gray-900">강의실 날짜/시간 선택</h3>
                <button id="close-classroom-select-date-modal-btn" class="text-gray-500 hover:text-gray-700 text-2xl font-semibold">
                    &times;
                </button>
            </div>
            <div class="mb-4">
                <label class="block text-gray-700 text-sm font-bold mb-2">선택 강의실:</label>
                <p id="classroom-modal-name" class="text-lg font-semibold text-blue-600"></p>
                <p id="classroom-modal-description" class="text-gray-700 text-sm"></p>
            </div>

            <div class="bg-gray-50 p-6 rounded-lg shadow-inner mb-6">
                <div class="flex justify-between items-center mb-4">
                    <button id="modal-prev-month-btn" class="p-2 rounded-full bg-blue-100 hover:bg-blue-200 text-blue-700 font-bold">‹</button>
                    <h3 id="modal-month-year-display" class="text-xl font-bold text-gray-800"></h3>
                    <button id="modal-next-month-btn" class="p-2 rounded-full bg-blue-100 hover:bg-blue-200 text-blue-700 font-bold">›</button>
                </div>
                <div class="grid grid-cols-7 gap-2 text-center font-semibold text-gray-700 mb-2">
                    <div>일</div><div>월</div><div>화</div><div>수</div><div>목</div><div>금</div><div>토</div>
                </div>
                <div id="modal-calendar-grid" class="grid grid-cols-7 gap-2">
                    <!-- Calendar days will be rendered here by JS -->
                </div>
            </div>

            <div id="modal-time-slots" class="p-4 bg-white rounded-lg shadow-inner">
                <!-- Time slots for selected classroom and date will be displayed here by JS -->
            </div>
        </div>
    </div>

    <!-- Custom Message Box -->
    <div id="message-box" class="hidden">
        <span id="message-text"></span>
    </div>
</body>
</html>
