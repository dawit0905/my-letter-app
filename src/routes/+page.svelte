<!--
  src/routes/+page.svelte

  '편지함 보기' 탭이 추가된 버전입니다.
  - Firestore에서 작성된 편지 목록을 불러와 보여줍니다.
  - 최신순으로 편지를 정렬합니다.
  - 장병을 선택하여 해당 장병의 편지만 필터링해서 볼 수 있습니다.
-->
<script>
    import { onMount } from 'svelte';
    import { initializeApp } from 'firebase/app';
    import { getFirestore, collection, getDocs, addDoc, serverTimestamp, query, orderBy } from 'firebase/firestore';

    // --- Firebase 설정 (환경 변수 사용) ---
    const firebaseConfig = {
        apiKey: import.meta.env.VITE_API_KEY,
        authDomain: import.meta.env.VITE_AUTH_DOMAIN,
        projectId: import.meta.env.VITE_PROJECT_ID,
        storageBucket: import.meta.env.VITE_STORAGE_BUCKET,
        messagingSenderId: import.meta.env.VITE_MESSAGING_SENDER_ID,
        appId: import.meta.env.VITE_APP_ID
    };

    // Firebase 앱 초기화
    const app = initializeApp(firebaseConfig);
    const db = getFirestore(app);

    // --- 상태 변수 ---
    let activeView = 'write'; // 현재 활성화된 뷰 ('write' 또는 'mailbox')
    let writeFlowPage = 'list'; // 편지 쓰기 플로우 내 현재 페이지 ('list' 또는 'write')
    let mailboxFlowPage = 'list'; // 편지함 플로우 내 현재 페이지 ('list' 또는 'letters')

    let soldiers = []; // 장병 목록
    let letters = []; // 전체 편지 목록

    let isLoadingSoldiers = true;
    let isLoadingLetters = false;

    let selectedSoldierForWrite = null; // 편지 쓸 때 선택한 장병
    let selectedSoldierForMailbox = null; // 편지함에서 선택한 장병
    let author = '';
    let message = '';
    let notification = '';

    // --- 데이터 로직 ---

    // 장병 목록 불러오기
    onMount(async () => {
        try {
            const querySnapshot = await getDocs(collection(db, "soldiers"));
            soldiers = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        } catch (error) {
            console.error("장병 목록 로딩 오류:", error);
            showNotification("장병 목록을 불러오는데 실패했습니다.");
        } finally {
            isLoadingSoldiers = false;
        }
    });

    // 편지 목록 불러오기
    async function fetchLetters() {
        // 이미 로딩된 데이터가 있으면 다시 불러오지 않음
        if (letters.length > 0) return;

        isLoadingLetters = true;
        try {
            const q = query(collection(db, "letters"), orderBy("createdAt", "desc"));
            const querySnapshot = await getDocs(q);
            letters = querySnapshot.docs.map(doc => {
                const data = doc.data();
                return {
                    id: doc.id,
                    ...data,
                    createdAt: data.createdAt ? data.createdAt.toDate() : new Date()
                };
            });
        } catch (error)
        {
            console.error("편지 목록 로딩 오류:", error);
            showNotification("편지 목록을 불러오는데 실패했습니다.");
        } finally {
            isLoadingLetters = false;
        }
    }

    // 편지 보내기
    async function handleSubmit() {
        if (!author.trim() || !message.trim()) {
            showNotification('작성자와 메시지를 모두 입력해주세요.');
            return;
        }
        try {
            const newLetter = {
                to: selectedSoldierForWrite.name,
                from: author,
                message: message,
                createdAt: serverTimestamp()
            };
            await addDoc(collection(db, "letters"), newLetter);

            // 편지 보내기 성공 후, 로컬 letters 배열에도 새 편지를 추가하여
            // 편지함으로 이동했을 때 바로 볼 수 있게 함
            letters = [{ ...newLetter, createdAt: new Date() }, ...letters];

            showNotification(`${selectedSoldierForWrite.name}님에게 편지를 성공적으로 보냈습니다!`, 'success');
            goBackToWriteList();
        } catch (error) {
            console.error("편지 저장 오류: ", error);
            showNotification("편지 전송에 실패했습니다. 다시 시도해주세요.");
        }
    }

    // --- UI 로직 ---

    function handleSelectSoldierForWrite(soldier) {
        selectedSoldierForWrite = soldier;
        writeFlowPage = 'write';
    }

    function goBackToWriteList() {
        writeFlowPage = 'list';
        selectedSoldierForWrite = null;
        author = '';
        message = '';
    }

    function handleSelectSoldierForMailbox(soldier) {
        selectedSoldierForMailbox = soldier;
        mailboxFlowPage = 'letters';
    }

    function goBackToMailboxList() {
        mailboxFlowPage = 'list';
        selectedSoldierForMailbox = null;
    }

    function changeView(view) {
        activeView = view;
        if (view === 'mailbox') {
            fetchLetters();
        }
    }

    function showNotification(msg, type = 'error') {
        notification = { msg, type };
        setTimeout(() => { notification = ''; }, 3000);
    }

    function formatDate(date) {
        if (!(date instanceof Date)) return '';
        return date.toLocaleDateString('ko-KR', { year: 'numeric', month: 'long', day: 'numeric' });
    }

    // 선택된 장병의 편지만 필터링하는 반응형 변수
    $: filteredLetters = selectedSoldierForMailbox
        ? letters.filter(letter => letter.to === selectedSoldierForMailbox.name)
        : [];

</script>

<main>
    <!-- 상단 네비게이션 -->
    <nav class="main-nav">
        <button on:click={() => changeView('write')} class:active={activeView === 'write'}>
            💌 편지 쓰기
        </button>
        <button on:click={() => changeView('mailbox')} class:active={activeView === 'mailbox'}>
            📬 편지함 보기
        </button>
    </nav>

    <!-- 편지 쓰기 뷰 -->
    {#if activeView === 'write'}
        <div class="page-container">
            {#if writeFlowPage === 'list'}
                <h1 class="title">누구에게 편지를 보낼까요?</h1>
                <p class="subtitle">편지를 전달할 장병을 선택해주세요.</p>
                {#if isLoadingSoldiers}
                    <p class="loading-text">장병 목록을 불러오는 중...</p>
                {:else}
                    <ul class="soldier-list">
                        {#each soldiers as soldier (soldier.id)}
                            <li on:click={() => handleSelectSoldierForWrite(soldier)}>
                                <span>{soldier.name}</span><span class="arrow">→</span>
                            </li>
                        {/each}
                    </ul>
                {/if}
            {/if}

            {#if writeFlowPage === 'write'}
                <div class="post-it">
                    <p class="recipient">To. {selectedSoldierForWrite.name}</p>
                    <textarea bind:value={message} class="message-input" placeholder="따뜻한 응원의 메시지를..."></textarea>
                    <input bind:value={author} type="text" class="author-input" placeholder="보내는 사람 이름"/>
                </div>
                <div class="button-group">
                    <button class="button secondary" on:click={goBackToWriteList}>뒤로가기</button>
                    <button class="button primary" on:click={handleSubmit}>편지 보내기</button>
                </div>
            {/if}
        </div>
    {/if}

    <!-- 편지함 보기 뷰 -->
    {#if activeView === 'mailbox'}
        <div class="page-container">
            {#if mailboxFlowPage === 'list'}
                <h1 class="title">누구의 편지함을 볼까요?</h1>
                <p class="subtitle">편지함을 보고 싶은 장병을 선택해주세요.</p>
                {#if isLoadingSoldiers}
                    <p class="loading-text">장병 목록을 불러오는 중...</p>
                {:else}
                    <ul class="soldier-list">
                        {#each soldiers as soldier (soldier.id)}
                            <li on:click={() => handleSelectSoldierForMailbox(soldier)}>
                                <span>{soldier.name}</span><span class="arrow">→</span>
                            </li>
                        {/each}
                    </ul>
                {/if}
            {/if}

            {#if mailboxFlowPage === 'letters'}
                <div class="mailbox-header">
                    <button class="back-button" on:click={goBackToMailboxList}>← 목록으로</button>
                    <h1 class="title small">{selectedSoldierForMailbox.name}의 편지함</h1>
                </div>
                {#if isLoadingLetters}
                    <p class="loading-text">편지를 불러오는 중...</p>
                {:else if filteredLetters.length === 0}
                    <p class="loading-text">아직 도착한 편지가 없어요.</p>
                {:else}
                    <div class="letter-grid">
                        {#each filteredLetters as letter (letter.id)}
                            <div class="letter-card">
                                <p class="letter-recipient">To. {letter.to}</p>
                                <p class="letter-message">"{letter.message}"</p>
                                <div class="letter-footer">
                                    <span class="letter-author">From. {letter.from}</span>
                                    <span class="letter-date">{formatDate(letter.createdAt)}</span>
                                </div>
                            </div>
                        {/each}
                    </div>
                {/if}
            {/if}
        </div>
    {/if}

    <!-- 알림 메시지 -->
    {#if notification}
        <div class="notification" class:success={notification.type === 'success'}>
            {notification.msg}
        </div>
    {/if}
</main>

<style>
    /* (기존 스타일 일부 재사용 및 추가) */
    @import url('https://fonts.googleapis.com/css2?family=Nanum+Pen+Script&display=swap');
    @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;500;700&display=swap');
    :global(body) { background-color: #f0f4f8; font-family: 'Noto Sans KR', sans-serif; display: flex; justify-content: center; align-items: flex-start; min-height: 100vh; margin: 0; padding-top: 2rem; }
    main { width: 100%; max-width: 480px; padding: 1rem; position: relative; }
    .page-container { background-color: white; border-radius: 16px; padding: 2rem; box-shadow: 0 10px 25px rgba(0, 0, 0, 0.05); animation: fadeIn 0.5s ease-out; margin-top: 1.5rem; }
    @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
    .title { font-size: 1.75rem; font-weight: 700; color: #1e293b; text-align: center; margin-bottom: 0.5rem; }
    .title.small { font-size: 1.5rem; margin: 0; flex-grow: 1; }
    .subtitle { font-size: 1rem; color: #64748b; text-align: center; margin-bottom: 2rem; }
    .loading-text { text-align: center; color: #64748b; padding: 2rem 0; }

    .main-nav { display: flex; background-color: #e2e8f0; border-radius: 12px; padding: 0.5rem; }
    .main-nav button { flex: 1; border: none; background: none; padding: 0.75rem 1rem; border-radius: 8px; font-size: 1rem; font-weight: 700; cursor: pointer; transition: all 0.2s ease; color: #475569; }
    .main-nav button.active { background-color: white; color: #4f46e5; box-shadow: 0 2px 10px rgba(0,0,0,0.08); }

    .soldier-list { list-style: none; padding: 0; margin: 0; }
    .soldier-list li { background-color: #f8fafc; padding: 1rem 1.25rem; border-radius: 12px; margin-bottom: 0.75rem; cursor: pointer; transition: all 0.2s ease-in-out; display: flex; justify-content: space-between; align-items: center; font-weight: 500; color: #334155; border: 1px solid #e2e8f0; }
    .soldier-list li:hover { transform: translateY(-2px); box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08); background-color: #eef2ff; border-color: #818cf8; }
    .soldier-list .arrow { font-size: 1.25rem; color: #818cf8; }

    .post-it { background-color: #fffde7; padding: 1.5rem; border-radius: 8px; box-shadow: 5px 5px 15px rgba(0, 0, 0, 0.1); transform: rotate(-2deg); font-family: 'Nanum Pen Script', cursive; margin-bottom: 2rem; }
    .recipient { font-size: 2rem; color: #333; margin: 0; border-bottom: 1px dashed rgba(0,0,0,0.1); padding-bottom: 0.5rem; margin-bottom: 1rem; }
    .message-input { width: 100%; height: 180px; background: transparent; border: none; resize: none; font-family: 'Nanum Pen Script', cursive; font-size: 1.8rem; line-height: 1.6; color: #444; }
    .author-input { width: 100%; background: transparent; border: none; text-align: right; font-family: 'Nanum Pen Script', cursive; font-size: 1.8rem; color: #444; margin-top: 1rem; }

    .button-group { display: flex; gap: 0.75rem; }
    .button { flex-grow: 1; border: none; padding: 0.8rem 1rem; border-radius: 12px; font-size: 1rem; font-weight: 700; cursor: pointer; transition: all 0.2s ease-in-out; }
    .button.primary { background-color: #4f46e5; color: white; }
    .button.secondary { background-color: #e2e8f0; color: #475569; }

    /* --- 겹침 문제 해결을 위한 수정 --- */
    .mailbox-header { display: flex; align-items: center; margin-bottom: 2rem; gap: 0.5rem; }
    .back-button { background: none; border: none; font-size: 1rem; font-weight: 500; color: #64748b; cursor: pointer; padding: 0.5rem; white-space: nowrap; }

    .letter-grid { display: grid; gap: 1rem; }
    .letter-card { background-color: #f8fafc; border: 1px solid #e2e8f0; border-radius: 12px; padding: 1.5rem; display: flex; flex-direction: column; }
    .letter-recipient { font-weight: 700; color: #334155; margin: 0 0 0.5rem; }
    .letter-message { flex-grow: 1; font-family: 'Nanum Pen Script', cursive; font-size: 1.5rem; line-height: 1.5; color: #475569; background-color: #fff; padding: 1rem; border-radius: 8px; margin: 0 0 1rem; white-space: pre-wrap; }
    .letter-footer { display: flex; justify-content: space-between; align-items: center; font-size: 0.875rem; color: #64748b; }
    .letter-author { font-weight: 500; }

    .notification { position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%); padding: 1rem 1.5rem; border-radius: 8px; background-color: #f87171; color: white; font-weight: 500; box-shadow: 0 4px 15px rgba(0,0,0,0.1); animation: slideIn 0.3s ease-out; }
    .notification.success { background-color: #4ade80; }
    @keyframes slideIn { from { opacity: 0; transform: translate(-50%, 20px); } to { opacity: 1; transform: translate(-50%, 0); } }
</style>
