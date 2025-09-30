<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>중학교 신입생 배정 거리 계산 웹 앱</title>
    <!-- Tailwind CSS 로드 -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- 폰트 설정 -->
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7f9fb;
        }
        .container {
            max-width: 1200px;
        }
        .header {
            background-color: #3b82f6; /* Blue-500 */
        }
    </style>
</head>
<body class="min-h-screen flex flex-col items-center">

    <!-- Firebase SDK 로드 -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        
        // Firestore 로깅 설정 (디버깅 목적)
        setLogLevel('Debug');

        // 전역 변수가 정의되어 있는지 확인 (Canvas 환경)
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let app, db, auth;
        let globalUserId = null;

        if (Object.keys(firebaseConfig).length > 0) {
            app = initializeApp(firebaseConfig);
            db = getFirestore(app);
            auth = getAuth(app);
            window.db = db; // 전역 접근 가능하도록 설정
            window.auth = auth;

            onAuthStateChanged(auth, (user) => {
                if (user) {
                    globalUserId = user.uid;
                } else {
                    globalUserId = crypto.randomUUID();
                }
                document.getElementById('user-id-display').textContent = 'User ID: ' + globalUserId.substring(0, 8) + '...';
                // 인증 준비 완료 후 로직 실행 (여기서는 파일 업로드가 메인 기능이므로 인증 후 특별한 데이터 로드는 없음)
            });

            async function initializeAuth() {
                try {
                    if (initialAuthToken) {
                        await signInWithCustomToken(auth, initialAuthToken);
                    } else {
                        await signInAnonymously(auth);
                    }
                } catch (error) {
                    console.error("Firebase Auth Error:", error);
                    // 익명 로그인 실패 시에도 임시 ID 설정
                    globalUserId = crypto.randomUUID();
                    document.getElementById('user-id-display').textContent = 'User ID: ' + globalUserId.substring(0, 8) + '... (Anon Fail)';
                }
            }
            initializeAuth();
        } else {
            console.warn("Firebase config not available. Running without persistent storage/auth.");
        }
    </script>
    
    <div class="header w-full text-white shadow-lg p-6 mb-8">
        <div class="container mx-auto">
            <h1 class="text-3xl font-bold tracking-tight">🏫 중학교 배정 거리 계산기</h1>
            <p id="user-id-display" class="text-sm opacity-75 mt-1"></p>
        </div>
    </div>

    <div class="container mx-auto px-4 w-full">
        <main class="bg-white p-8 rounded-xl shadow-2xl space-y-8">
            
            <!-- 안내 및 설정 섹션 -->
            <section class="border-b pb-6">
                <h2 class="text-2xl font-semibold text-gray-800 mb-4">📍 사용 준비 및 안내</h2>
                <div id="setup-instructions" class="bg-red-100 border-l-4 border-red-500 text-red-800 p-4 rounded-lg space-y-3">
                    <p class="font-medium">
                        ✅ **Google Maps API 키가 설정되었습니다.**
                    </p>
                    <p class="text-sm font-bold text-red-800">
                        ❌ **API 활성화 오류 (ApiNotActivatedMapError) 해결을 위해 필독!**
                    </p>
                    <p class="text-sm">
                        새로운 키가 적용되었지만, 이 오류는 Google Cloud Console에서 **Geocoding API** 및 **Directions API**가 해당 키에 대해 **활성화(Enable)**되어 있지 않아 발생합니다. 
                        **이 두 API를 활성화해야만** 주소-좌표 변환 및 거리 계산이 정상적으로 동작합니다.
                    </p>
                    <p class="text-xs text-gray-700 mt-2">
                        API 활성화 후 새로고침하여 재시도해 주세요.
                    </p>
                </div>
            </section>

            <!-- 1. 거주지(신입생) 데이터 입력 섹션 -->
            <section class="space-y-4 border-b pb-6">
                <h2 class="text-2xl font-semibold text-gray-800">1️⃣ 거주지 데이터 입력 (성명, 주소)</h2>
                
                <!-- 입력 방식 선택 -->
                <div class="flex items-center space-x-4">
                    <label class="inline-flex items-center">
                        <input type="radio" name="residence-input-mode" value="file" checked class="residence-mode-radio form-radio text-blue-600 h-4 w-4">
                        <span class="ml-2 text-gray-700 font-medium">CSV 파일 업로드</span>
                    </label>
                    <label class="inline-flex items-center">
                        <input type="radio" name="residence-input-mode" value="manual" class="residence-mode-radio form-radio text-blue-600 h-4 w-4">
                        <span class="ml-2 text-gray-700 font-medium">직접 입력 (수기)</span>
                    </label>
                </div>
                
                <!-- 파일 업로드 필드 -->
                <div id="residence-file-input-container">
                    <input type="file" id="residence-data-file" accept=".csv" class="w-full text-sm text-gray-500
                        file:mr-4 file:py-2 file:px-4
                        file:rounded-full file:border-0
                        file:text-sm file:font-semibold
                        file:bg-blue-50 file:text-blue-700
                        hover:file:bg-blue-100 cursor-pointer"/>
                    <p class="text-xs text-gray-500 mt-1">CSV 형식: <code class="bg-gray-100 p-1 rounded">Name,Address</code></p>
                </div>

                <!-- 수기 입력 필드 -->
                <div id="residence-manual-input-container" class="hidden">
                    <textarea id="residence-manual-input" rows="5" class="w-full p-2 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500" placeholder="각 줄마다 '성명,주소' 형식으로 입력하세요.&#10;예: 홍길동,서울시 강남구 역삼동 123&#10;김철수,부산시 해운대구 우동 789"></textarea>
                    <p class="text-xs text-gray-500 mt-1">한 줄에 하나씩 `성명,주소` 형식으로 입력하세요.</p>
                </div>
            </section>
            
            <!-- 2. 학교 데이터 입력 섹션 -->
            <section class="space-y-4 border-b pb-6">
                <h2 class="text-2xl font-semibold text-gray-800">2️⃣ 중학교 데이터 입력 (학교명, 주소)</h2>
                <div id="school-manual-input-container">
                    <textarea id="school-manual-input" rows="3" class="w-full p-2 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500" placeholder="각 줄마다 '학교명,주소' 형식으로 입력하세요.&#10;예: 강남중학교,서울시 강남구 삼성동 456&#10;해운대중학교,부산시 해운대구 중동 101"></textarea>
                    <p class="text-xs text-gray-500 mt-1">한 줄에 하나씩 `학교명,주소` 형식으로 입력하세요.</p>
                </div>
            </section>

            <!-- 계산 실행 섹션 -->
            <section class="space-y-4">
                <h2 class="text-2xl font-semibold text-gray-800">🚀 거리 계산 실행</h2>
                
                <button id="calculate-button" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-lg transition duration-150 shadow-md disabled:bg-gray-400">
                    거리 계산 시작
                </button>
                
                <div id="status-message" class="p-3 text-center rounded-lg text-sm transition duration-300 hidden"></div>
                <div id="loading-indicator" class="hidden text-center">
                    <div class="animate-spin inline-block w-6 h-6 border-[3px] border-current border-t-transparent text-blue-500 rounded-full" role="status">
                        <span class="sr-only">Loading...</span>
                    </div>
                    <p class="text-blue-600 mt-2">데이터를 처리 중입니다. (API 호출이 많아 시간이 다소 소요될 수 있습니다.)</p>
                </div>
            </section>

            <!-- 결과 표시 섹션 -->
            <section>
                <h2 class="text-2xl font-semibold text-gray-800 mb-4 border-t pt-6 mt-6">📊 계산 결과 (최단 거리)</h2>
                <div id="results-table-container" class="overflow-x-auto rounded-lg shadow-inner">
                    <table id="results-table" class="min-w-full divide-y divide-gray-200">
                        <thead class="bg-gray-50">
                            <tr>
                                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">No.</th>
                                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">성명</th>
                                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">거주지 주소</th>
                                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">가장 가까운 학교명</th>
                                <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">최단 거리 (km)</th>
                            </tr>
                        </thead>
                        <tbody id="results-body" class="bg-white divide-y divide-gray-200">
                            <!-- 결과 행이 여기에 삽입됩니다. -->
                            <tr><td colspan="5" class="px-6 py-4 whitespace-nowrap text-sm text-gray-500 text-center">데이터를 입력하고 계산 버튼을 눌러주세요.</td></tr>
                        </tbody>
                    </table>
                </div>
                
                <button id="download-button" class="mt-4 bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded-lg transition duration-150 shadow-md hidden disabled:bg-gray-400">
                    결과 CSV 다운로드
                </button>
            </section>

        </main>
    </div>

    <!-- Google Maps API 로드를 위한 스크립트 (Defer 로드) -->
    <script async defer id="google-maps-script"></script>

    <script>
        // ==============================================================================
        // Google Maps API 키 (이전에 제공된 키 사용)
        // ==============================================================================
        const GOOGLE_API_KEY = "AIzaSyD7nOPYCIbHnrtdkEs2Kpjx72VuaWEEo-I"; 

        let directionsService;
        let geocoder;
        let resultsData = []; // 최종 결과 저장소

        const residenceModeRadios = document.querySelectorAll('.residence-mode-radio');
        const residenceFileInputContainer = document.getElementById('residence-file-input-container');
        const residenceManualInputContainer = document.getElementById('residence-manual-input-container');
        const residenceFile = document.getElementById('residence-data-file');
        const residenceManualInput = document.getElementById('residence-manual-input');
        const schoolManualInput = document.getElementById('school-manual-input');
        
        const calculateButton = document.getElementById('calculate-button');
        const downloadButton = document.getElementById('download-button');
        const statusMessage = document.getElementById('status-message');
        const loadingIndicator = document.getElementById('loading-indicator');
        const resultsBody = document.getElementById('results-body');

        // ==== UI 상호작용 및 초기화 ====

        // 거주지 입력 방식 토글
        residenceModeRadios.forEach(radio => {
            radio.addEventListener('change', () => {
                const mode = document.querySelector('input[name="residence-input-mode"]:checked').value;
                if (mode === 'file') {
                    residenceFileInputContainer.classList.remove('hidden');
                    residenceManualInputContainer.classList.add('hidden');
                } else {
                    residenceFileInputContainer.classList.add('hidden');
                    residenceManualInputContainer.classList.remove('hidden');
                }
            });
        });

        // Google Maps API 스크립트 동적 로드
        function loadGoogleMapsScript() {
            const script = document.getElementById('google-maps-script');
            script.src = `https://maps.googleapis.com/maps/api/js?key=${GOOGLE_API_KEY}&libraries=geometry&callback=initMap`;
            script.onerror = () => {
                showStatus('Google Maps API 로드 실패. 키가 올바른지 확인하거나 필수 API가 활성화되어 있는지 확인하세요.', 'error');
                calculateButton.disabled = true;
            };
        }

        // Google Maps 초기화 콜백 (API 로드 후 자동으로 호출됨)
        window.initMap = function() {
            directionsService = new google.maps.DirectionsService();
            geocoder = new google.maps.Geocoder();
            calculateButton.disabled = false;
            
            // API 활성화 오류가 있을 경우에도 사용자에게 명확히 알리기 위해 상태 메시지 강제 업데이트
            const setupDiv = document.getElementById('setup-instructions');
            setupDiv.classList.remove('bg-green-50', 'border-green-400', 'text-green-800');
            setupDiv.classList.add('bg-red-100', 'border-red-500', 'text-red-800');
            
            showStatus('Google Maps API가 준비되었습니다. (API 활성화 상태 확인 필요)', 'info');

        };

        // 상태 메시지 표시
        function showStatus(message, type = 'info') {
            statusMessage.textContent = message;
            statusMessage.classList.remove('hidden', 'bg-red-100', 'text-red-700', 'bg-green-100', 'text-green-700', 'bg-blue-100', 'text-blue-700');
            if (type === 'error') {
                statusMessage.classList.add('bg-red-100', 'text-red-700');
            } else if (type === 'success') {
                statusMessage.classList.add('bg-green-100', 'text-green-700');
            } else {
                statusMessage.classList.add('bg-blue-100', 'text-blue-700');
            }
        }

        // 로딩 상태 토글
        function toggleLoading(isLoading) {
            loadingIndicator.classList.toggle('hidden', !isLoading);
            calculateButton.disabled = isLoading;
            calculateButton.textContent = isLoading ? '계산 중...' : '거리 계산 시작';
        }

        // CSV/Manual Text 파서 (Name, Address / SchoolName, Address)
        function parseTextData(text, headers) {
            const lines = text.trim().split('\n').filter(line => line.trim() !== '');
            const data = [];
            lines.forEach(line => {
                const values = line.split(',').map(v => v.trim());
                if (values.length >= 2) {
                    const row = {};
                    row[headers[0]] = values[0]; // Name or SchoolName
                    row[headers[1]] = values.slice(1).join(','); // Address (주소에 쉼표가 있을 경우 대비)
                    data.push(row);
                }
            });
            return data;
        }

        // 거주지 데이터 파싱 (파일 또는 수기)
        async function parseResidenceInput() {
            const mode = document.querySelector('input[name="residence-input-mode"]:checked').value;
            let rawText = '';
            
            if (mode === 'file') {
                if (residenceFile.files.length === 0) throw new Error("CSV 파일을 선택해주세요.");
                const file = residenceFile.files[0];
                rawText = await new Promise((resolve, reject) => {
                    const reader = new FileReader();
                    reader.onload = (e) => resolve(e.target.result);
                    reader.onerror = (e) => reject(new Error("파일 읽기 오류"));
                    // 파일 인코딩 문제로 데이터가 깨질 경우, 'euc-kr' 대신 'utf-8'로 시도해 볼 수 있습니다.
                    reader.readAsText(file, 'euc-kr'); 
                });
                // CSV 파일의 첫 줄(헤더) 무시
                rawText = rawText.split('\n').slice(1).join('\n'); 
            } else {
                rawText = residenceManualInput.value;
                if (rawText.trim() === '') throw new Error("거주지 주소를 직접 입력해주세요.");
            }
            
            return parseTextData(rawText, ['Name', 'Address']);
        }

        // 학교 데이터 파싱 (수기 입력)
        function parseSchoolInput() {
            const rawText = schoolManualInput.value;
            if (rawText.trim() === '') throw new Error("중학교 주소를 입력해주세요.");
            
            return parseTextData(rawText, ['SchoolName', 'SchoolAddress']);
        }

        // 지수 백오프를 사용한 비동기 API 호출 래퍼 (기존과 동일)
        async function withExponentialBackoff(apiCall, maxRetries = 5, initialDelay = 1000) {
            for (let attempt = 0; attempt < maxRetries; attempt++) {
                try {
                    const result = await apiCall();
                    return result;
                } catch (error) {
                    // ApiNotActivatedMapError가 발생하는 경우, 이 오류를 먼저 감지하여 사용자에게 더 명확한 메시지를 전달할 수 있도록 처리합니다.
                    if (error.message && error.message.includes('ApiNotActivatedMapError')) {
                        throw new Error("API_NOT_ACTIVATED");
                    }
                    if (error.message.includes('Directions API 오류: ZERO_RESULTS')) {
                        throw error;
                    }
                    if (attempt === maxRetries - 1) {
                        throw error;
                    }
                    const delay = initialDelay * Math.pow(2, attempt) + Math.random() * 1000;
                    // API 호출 실패 시 콘솔 로깅은 유지
                    // console.warn(`API 호출 실패 (시도 ${attempt + 1}/${maxRetries}). ${Math.round(delay/1000)}초 후 재시도...`, error);
                    await new Promise(resolve => setTimeout(resolve, delay));
                }
            }
        }

        // 주소로 좌표를 가져오는 함수 (Geocoding) - 기존과 동일
        async function getCoordinates(address) {
            return withExponentialBackoff(async () => {
                const response = await geocoder.geocode({ 'address': address });
                if (response.results.length === 0) {
                    throw new Error("Geocoding: 주소를 찾을 수 없습니다.");
                }
                const location = response.results[0].geometry.location;
                return { lat: location.lat(), lng: location.lng() };
            });
        }

        // 좌표 쌍 간의 실제 도로 거리를 계산하는 함수 (Directions) - 기존과 동일
        async function getDrivingDistance(origin, destination) {
            return withExponentialBackoff(() => {
                return new Promise((resolve, reject) => {
                    directionsService.route({
                        origin: origin,
                        destination: destination,
                        // 현재 DRIVING (차량) 기준 최단 경로 사용. 도보 기준이 필요하면 WALKING으로 변경 가능.
                        travelMode: google.maps.TravelMode.DRIVING, 
                    }, (response, status) => {
                        if (status === 'OK') {
                            const route = response.routes[0].legs[0];
                            const distanceKm = route.distance.value / 1000; 
                            resolve(distanceKm);
                        } else {
                            // API 활성화 오류가 아닌 다른 Directions 오류 처리
                            reject(new Error(`Directions API 오류: ${status}`));
                        }
                    });
                });
            });
        }

        // 병렬 처리를 위한 배치 계산 함수 (최적화: 중간 UI 업데이트 제거)
        async function calculateDistanceBatches(residences, schools, getCachedCoordinates) {
            const MAX_CONCURRENT_API_CALLS = 5; // 동시 API 호출 제한
            const totalTasks = residences.length * schools.length;
            let completedTasks = 0;
            const closestSchoolMap = new Map();

            // 모든 거주지와 학교 쌍을 작업 목록으로 생성
            const tasks = [];
            for (const residence of residences) {
                for (const school of schools) {
                    tasks.push({ residence, school });
                }
            }

            // 작업을 일괄 처리할 청크 사이즈 결정 (최대 동시 호출 건수)
            const chunkSize = MAX_CONCURRENT_API_CALLS; 

            for (let i = 0; i < tasks.length; i += chunkSize) {
                const chunk = tasks.slice(i, i + chunkSize);
                const chunkPromises = chunk.map(task => 
                    (async () => {
                        const { residence, school } = task;
                        let residenceName = residence.Name;
                        
                        let currentBestResult = closestSchoolMap.get(residenceName) || {
                            residenceName: residenceName,
                            residenceAddress: residence.Address,
                            closestSchool: { name: 'N/A', address: 'N/A' },
                            minDistanceKm: Infinity,
                            status: '처리 중'
                        };

                        try {
                            const residenceCoords = await getCachedCoordinates(residence.Address);
                            if (!residenceCoords) throw new Error("거주지 주소 변환 불가");
                            
                            const schoolCoords = await getCachedCoordinates(school.SchoolAddress);
                            if (!schoolCoords) throw new Error(`학교 (${school.SchoolName}) 주소 변환 불가`);

                            const schoolDistanceKm = await getDrivingDistance(residenceCoords, schoolCoords);

                            // 현재 계산된 거리가 해당 거주지의 최단 거리보다 짧으면 업데이트
                            if (schoolDistanceKm < currentBestResult.minDistanceKm) {
                                currentBestResult.minDistanceKm = schoolDistanceKm;
                                currentBestResult.closestSchool = {
                                    name: school.SchoolName,
                                    address: school.SchoolAddress
                                };
                                currentBestResult.status = '최단 거리 업데이트';
                            }

                        } catch (e) {
                            if (e.message === "API_NOT_ACTIVATED") throw e;
                            
                            // Geocoding 오류이거나 Directions 오류일 경우, 현재 학교와의 계산만 실패로 처리하고 계속 진행
                            if (e.message.includes('거주지 주소 변환 불가') && currentBestResult.status === '처리 중') {
                                currentBestResult.status = `거주지 오류: ${e.message.substring(13)}`; // 오류 메시지 축약
                            } else if (e.message.includes('학교') || e.message.includes('Directions API 오류')) {
                                // 학교별 오류는 최단 거리 갱신에 영향을 주지 않으므로 상태를 변경하지 않음
                            } else {
                                console.error(`Unexpected error for ${residence.Name}: ${e.message}`);
                            }
                        } finally {
                            completedTasks++;
                            // 매번 최단 결과를 맵에 저장
                            closestSchoolMap.set(residenceName, currentBestResult); 
                            
                            // UI 업데이트를 매 배치마다 하지 않고, 상태 메시지만 업데이트하여 속도 개선
                            if (completedTasks % chunkSize === 0 || completedTasks === totalTasks) {
                                showStatus(`처리 중: ${completedTasks}/${totalTasks} 건 완료 (${Math.floor((completedTasks / totalTasks) * 100)}%)`, 'info');
                            }
                        }
                    })()
                );
                // 현재 청크의 모든 Promise가 완료될 때까지 기다립니다.
                try {
                    await Promise.all(chunkPromises);
                } catch (e) {
                    if (e.message === "API_NOT_ACTIVATED") throw e;
                    console.error("Batch error detected, continuing to next batch if possible.");
                }
            }

            return Array.from(closestSchoolMap.values());
        }

        // 결과 테이블을 업데이트하는 함수 (최소 필드만 남기고 '상태' 제거)
        function updateResultsTable(closestSchoolMap) {
            resultsBody.innerHTML = '';
            resultsData = [];
            let finalIndex = 0;

            for (const [name, result] of closestSchoolMap.entries()) {
                finalIndex++;
                
                const isSuccess = result.minDistanceKm !== Infinity; 
                const distanceDisplay = isSuccess ? `${result.minDistanceKm.toFixed(2)} km` : 'N/A';
                const statusDisplay = isSuccess ? '성공 (최단 거리 확인)' : (result.status === '처리 중' ? '모든 학교 계산 실패' : result.status);
                const statusColor = isSuccess ? 'text-green-600' : 'text-red-600';
                
                // 결과 테이블 업데이트 (최소 필드만 출력)
                let row = resultsBody.insertRow();
                row.classList.add(isSuccess ? 'hover:bg-green-50' : 'bg-red-50');
                row.innerHTML = `
                    <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">${finalIndex}</td>
                    <td class="px-6 py-4 whitespace-nowrap text-sm font-bold text-gray-900">${result.residenceName}</td>
                    <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">${result.residenceAddress}</td>
                    <td class="px-6 py-4 whitespace-nowrap text-sm font-bold ${isSuccess ? 'text-blue-600' : 'text-gray-500'}">${result.closestSchool.name}</td>
                    <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900 font-extrabold distance-cell">${distanceDisplay}</td>
                    <!-- '상태' 컬럼 제거됨 -->
                `;

                // 결과 데이터 저장 (다운로드용)
                resultsData.push({
                    No: finalIndex,
                    Name: result.residenceName,
                    ResidenceAddress: result.residenceAddress,
                    ClosestSchoolName: result.closestSchool.name,
                    ClosestSchoolAddress: result.closestSchool.address,
                    DistanceKm: isSuccess ? result.minDistanceKm.toFixed(2) : 'N/A',
                    Status: statusDisplay // 상태는 데이터에는 남겨두지만 테이블에는 표시하지 않음
                });
            }
        }


        // ==== 메인 처리 로직 (최단 거리 계산 로직으로 변경) ====

        async function processCalculation() {
            resultsBody.innerHTML = '<tr><td colspan="5" class="px-6 py-4 whitespace-nowrap text-sm text-gray-500 text-center text-blue-600 font-medium">데이터를 파싱하는 중...</td></tr>';
            downloadButton.classList.add('hidden');
            resultsData = [];
            toggleLoading(true);

            let residences;
            let schools;

            try {
                // 1. 데이터 파싱
                residences = await parseResidenceInput();
                schools = parseSchoolInput();

                if (residences.length === 0) throw new Error("처리할 거주지 데이터가 없습니다.");
                if (schools.length === 0) throw new Error("처리할 중학교 데이터가 없습니다.");
                
                showStatus(`총 ${residences.length}명의 거주지와 ${schools.length}개의 학교에 대해 ${residences.length * schools.length}건의 거리를 계산합니다. (거주지별 최단 거리 확인)`, 'info');

            } catch (error) {
                console.error("데이터 파싱 오류:", error);
                showStatus(`데이터 입력 오류: ${error.message}`, 'error');
                toggleLoading(false);
                resultsBody.innerHTML = '<tr><td colspan="5" class="px-6 py-4 whitespace-nowrap text-sm text-gray-500 text-center text-red-600 font-bold">계산 오류: 데이터를 확인해 주세요.</td></tr>';
                return;
            }

            // 2. Geocoding 캐시 (중복 호출 최소화)
            const geocodeCache = new Map();
            const getCachedCoordinates = async (address) => {
                if (!geocodeCache.has(address)) {
                    try {
                         // Geocoding 호출
                         geocodeCache.set(address, await getCoordinates(address));
                    } catch (e) {
                        // API_NOT_ACTIVATED 오류 재전파
                        if (e.message === "API_NOT_ACTIVATED") throw e;
                        // 다른 오류는 null로 처리하여 거리 계산을 건너뛰도록 함
                        geocodeCache.set(address, null);
                        throw e; // Geocoding 오류 발생 시 throws
                    }
                }
                return geocodeCache.get(address);
            };

            // 3. 최단 거리 계산 실행 및 배치 처리
            try {
                const finalResults = await calculateDistanceBatches(residences, schools, getCachedCoordinates);
                
                // 최종 결과 테이블 업데이트 (모든 계산이 끝난 후 한 번만 실행)
                updateResultsTable(new Map(finalResults.map(r => [r.residenceName, r])));

            } catch (e) {
                if (e.message === "API_NOT_ACTIVATED") {
                    showStatus('치명적 오류: Geocoding/Directions API가 활성화되지 않았습니다. 콘솔을 확인하세요.', 'error');
                } else {
                    showStatus(`계산 중 치명적 오류 발생: ${e.message}`, 'error');
                    console.error("Critical error during calculation:", e);
                }
                toggleLoading(false);
                return;
            }
            
            toggleLoading(false);
            showStatus('모든 거주지별 최단 거리 계산이 완료되었습니다!', 'success');
            downloadButton.classList.remove('hidden');
        }

        // 결과 CSV 다운로드 함수 (최단 거리 결과에 맞게 업데이트)
        function downloadCSV() {
            if (resultsData.length === 0) return;

            // CSV 헤더: 가장 가까운 학교 정보를 포함하도록 구성
            let csvContent = "No.,성명,거주지 주소,가장 가까운 학교명,가장 가까운 학교 주소,최단 거리 (km),상태\n";
            resultsData.forEach(row => {
                const rowArray = [
                    row.No, 
                    `"${row.Name.replace(/"/g, '""')}"`,
                    `"${row.ResidenceAddress.replace(/"/g, '""')}"`, 
                    `"${row.ClosestSchoolName.replace(/"/g, '""')}"`, 
                    `"${row.ClosestSchoolAddress.replace(/"/g, '""')}"`,
                    row.DistanceKm, 
                    row.Status
                ];
                csvContent += rowArray.join(',') + "\n";
            });

            const blob = new Blob(["\uFEFF" + csvContent], { type: 'text/csv;charset=utf-8;' }); // BOM 추가로 한글 인코딩 문제 해결
            const link = document.createElement("a");
            const url = URL.createObjectURL(blob);
            link.setAttribute("href", url);
            link.setAttribute("download", "closest_school_results.csv"); // 파일명 변경
            link.style.visibility = 'hidden';
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }

        // ==== 이벤트 리스너 설정 ====

        calculateButton.addEventListener('click', () => {
            if (!directionsService) {
                showStatus('Google Maps API 로드를 기다려주세요.', 'error');
                return;
            }
            // API 활성화 오류가 발생하면, 재시도 없이 프로세스를 중단하고 사용자에게 오류 메시지를 표시하도록 try/catch 추가
            try {
                processCalculation();
            } catch (e) {
                if (e.message === "API_NOT_ACTIVATED") {
                    // processCalculation 내부에서 이미 토글 및 상태 표시가 이루어졌으므로 추가 작업 불필요
                    console.error("Critical API Activation Error detected.");
                }
            }
        });
        
        downloadButton.addEventListener('click', downloadCSV);

        // 앱 시작 시 Google Maps API 로드 시도
        window.onload = loadGoogleMapsScript;
    </script>
</body>
</html>
