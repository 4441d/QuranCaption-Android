<script lang="ts">
	import { onMount } from 'svelte';
	import { globalState } from '$lib/runes/main.svelte';
	import { ProjectEditorTabs } from '$lib/classes';
	import RecitersManager from '$lib/classes/Reciter';

	type Message = { role: 'user' | 'assistant'; text: string };
	type AssistantStatus = 'ready' | 'working' | 'success' | 'error';
	type PendingAction = { kind: 'caption' | 'export' | 'fps'; label: string; value?: number };

	let open = $state(false);
	let input = $state('');
	let isDragging = $state(false);
	let isResizing = $state(false);
	let isIdle = $state(false);
	let isTyping = $state(false);
	let status = $state<AssistantStatus>('ready');
	let position = $state({ right: 24, bottom: 84 });
	let size = $state(58);
	let pendingAction = $state<PendingAction | null>(null);
	let pressTimer: ReturnType<typeof setTimeout> | undefined;
	let idleTimer: ReturnType<typeof setTimeout> | undefined;
	let dragStart = { x: 0, y: 0, right: 24, bottom: 84 };
	let resizeStart = { x: 0, y: 0, size: 58 };
	let messages = $state<Message[]>([
		{
			role: 'assistant',
			text: 'مرحبًا. أستطيع فهم أوامر التطبيق وفتح الأدوات الموجودة. سأطلب موافقتك قبل تشغيل الكابتشن أو التصدير.'
		}
	]);

	function setTab(tab: ProjectEditorTabs): void {
		if (!globalState.currentProject) return;
		globalState.currentProject.projectEditorState.currentTab = tab;
	}

	function normalize(text: string): string {
		return text.trim().toLocaleLowerCase().normalize('NFD').replace(/[\u064B-\u065F\u0670]/g, '').replace(/[أإآ]/g, 'ا').replace(/ة/g, 'ه');
	}

	function resetIdle(): void {
		isIdle = false;
		if (idleTimer) clearTimeout(idleTimer);
		idleTimer = setTimeout(() => {
			if (!open && !isDragging && !isResizing) {
				isIdle = true;
				position = { ...position, bottom: 16 };
			}
		}, 2600);
	}

	function addAssistant(text: string, nextStatus: AssistantStatus = 'success'): void {
		messages = [...messages, { role: 'assistant', text }];
		status = nextStatus;
		resetIdle();
	}

	function requestAction(action: PendingAction): string {
		pendingAction = action;
		status = 'working';
		return `سأجهز ${action.label} الآن. اضغط «موافق وتنفيذ» إذا كان هذا قصدك، أو «إلغاء» للتراجع.`;
	}

	function confirmAction(): void {
		const action = pendingAction;
		if (!action) return;
		pendingAction = null;
		status = 'working';
		if (action.kind === 'caption') {
			setTab(ProjectEditorTabs.SubtitlesEditor);
			globalState.shared.openAutoSegmentationRequest += 1;
			addAssistant('فتحت معالج الكابتشن/التجزئة التلقائية في قسم الترجمات. اختر الصوت والإعدادات، ثم تابع من المعالج. إذا حدث فشل سيظهر لك سببه بدل إخفائه.', 'success');
			return;
		}
		if (action.kind === 'fps' && action.value) {
			setTab(ProjectEditorTabs.Export);
			globalState.getExportState.fps = action.value;
			addAssistant(`تم ضبط معدل الإطارات فعليًا على ${action.value} إطار/ثانية وفتحت شاشة التصدير للمراجعة.`, 'success');
			return;
		}
		setTab(ProjectEditorTabs.Export);
		addAssistant('فتحت شاشة التصدير. راجع المدى والاسم ثم اضغط زر التصدير من الشاشة؛ لا أبدأ تصديرًا طويلًا دون تأكيد نهائي منك.', 'success');
	}

	function cancelAction(): void {
		pendingAction = null;
		status = 'ready';
		addAssistant('تم الإلغاء. لم أغيّر المشروع ولم أبدأ أي عملية.', 'ready');
	}

	function answer(question: string): string {
		const q = normalize(question);
		if (!q) return 'اكتب طلبًا واضحًا مثل: اعمل caption للصوت، افتح البحث، اعرض الشيوخ، أو جهز التصدير.';
		if (q.includes('موافق') || q.includes('نفذ') || q.includes('نفّذ') || q === 'نعم') {
			if (pendingAction) {
				confirmAction();
				return 'حسنًا، أنفذ الخطوة التي اتفقنا عليها.';
			}
			return 'لا توجد عملية معلقة تحتاج موافقة.';
		}
		if (q.includes('الغ') || q.includes('إلغاء') || q.includes('الغاء')) {
			if (pendingAction) cancelAction();
			return 'ألغيت العملية المعلقة.';
		}
		if (q.includes('caption') || q.includes('كابتشن') || q.includes('كابشن') || q.includes('التقاط الصوت') || q.includes('ترجمة الصوت') || q.includes('حول الصوت')) {
			return requestAction({ kind: 'caption', label: 'توليد الكابتشن من الصوت' });
		}
		if (q.includes('صدّر') || q.includes('تصدير') || q.includes('export')) {
			return requestAction({ kind: 'export', label: 'تجهيز تصدير الفيديو' });
		}
		if (q.includes('60 فريم') || q.includes('60 اطار') || q.includes('60 إطار')) {
			return requestAction({ kind: 'fps', value: 60, label: 'ضبط التصدير على 60 إطارًا/ثانية' });
		}
		if (q.includes('30 فريم') || q.includes('30 اطار') || q.includes('30 إطار')) {
			return requestAction({ kind: 'fps', value: 30, label: 'ضبط التصدير على 30 إطارًا/ثانية' });
		}
		if (q.includes('شيخ') || q.includes('قارئ') || q.includes('reciter')) {
			const reciters = RecitersManager.getRecitersWithCustomOnes();
			const names = reciters.slice(0, 12).map((r) => r.label).join('، ');
			return `يوجد ${reciters.length} شيخًا/قارئًا متاحًا. من القائمة: ${names}${reciters.length > 12 ? '…' : ''}. لإضافة شيخ جديد أحتاج ملف الصوت أو رابطًا موثوقًا، وسأطلب تأكيدك قبل إضافته.`;
		}
		if (q.includes('بحث') || q.includes('ابحث') || q.includes('search')) {
			globalState.shared.projectSearch.openRequest += 1;
			return 'فتحت البحث داخل المشروع. اكتب كلمة البحث هناك، ويمكنك إغلاق المحادثة وسأترك البحث مفتوحًا.';
		}
		if (q.includes('ترجم') || q.includes('translation')) {
			setTab(ProjectEditorTabs.Translations);
			return 'فتحت قسم الترجمات. يمكنك إغلاق المحادثة ومتابعة التعديل بشكل طبيعي.';
		}
		if (q.includes('نمط') || q.includes('ستايل') || q.includes('style')) {
			setTab(ProjectEditorTabs.Style);
			return 'فتحت قسم النمط. لم أعدّل شيئًا تلقائيًا.';
		}
		if (q.includes('محرر الفيديو') || q.includes('محرر فيديو') || q.includes('فيديو') || q.includes('video')) {
			setTab(ProjectEditorTabs.VideoEditor);
			return 'فتحت محرر الفيديو. يمكنك متابعة العمل خارجه، والمحادثة لا توقف التطبيق.';
		}
		if (q.includes('السلام عليكم') || q.includes('سلام عليكم') || q.includes('مرحبا') || q.includes('اهلا')) {
			return 'وعليكم السلام ورحمة الله وبركاته. أنا جاهز لفهم طلبك داخل Quran Caption.';
		}
		if (q.includes('مساعدة') || q.includes('ماذا') || q.includes('help')) {
			return 'أفهم أوامر البحث، الكابتشن، الترجمات، الأنماط، الشيوخ، الفيديو والتصدير. أفتح الأدوات الموجودة فعلًا، ولا أدّعي تنفيذ عملية غير موجودة. العمليات التي تغيّر المشروع أو تبدأ تصديرًا تحتاج موافقتك.';
		}
		return 'أحتاج تحديد الجزء المقصود. هل تريد: محرر الفيديو، الترجمات، النمط، الشيوخ، البحث، الكابتشن من الصوت، أم التصدير؟ اكتب «افتح محرر الفيديو» أو «افتح النمط» وسأنقلك مباشرة.';
	}

	async function send(): Promise<void> {
		const text = input.trim();
		if (!text) return;
		status = 'working';
		messages = [...messages, { role: 'user', text }];
		input = '';
		isTyping = true;
		resetIdle();
		await new Promise((resolve) => setTimeout(resolve, 700));
		const reply = answer(text);
		isTyping = false;
		messages = [...messages, { role: 'assistant', text: reply }];
		if (status === 'working') status = 'success';
		resetIdle();
	}

	function startPointer(event: PointerEvent): void {
		if ((event.target as HTMLElement).closest('[data-resize]')) return;
		resetIdle();
		const target = event.currentTarget as HTMLElement;
		target.setPointerCapture(event.pointerId);
		pressTimer = setTimeout(() => {
			isDragging = true;
			dragStart = { x: event.clientX, y: event.clientY, ...position };
		}, 450);
	}

	function movePointer(event: PointerEvent): void {
		if (!isDragging) return;
		position = {
			right: Math.max(8, Math.min(window.innerWidth - size - 8, dragStart.right - (event.clientX - dragStart.x))),
			bottom: Math.max(8, Math.min(window.innerHeight - size - 8, dragStart.bottom + (event.clientY - dragStart.y)))
		};
		localStorage.setItem('quran-caption-assistant-position', JSON.stringify({ position, size }));
	}

	function endPointer(): void {
		if (pressTimer) clearTimeout(pressTimer);
		if (!isDragging && !isResizing) open = !open;
		isDragging = false;
		isResizing = false;
		resetIdle();
	}

	function startResize(event: PointerEvent): void {
		event.stopPropagation();
		resetIdle();
		isResizing = true;
		resizeStart = { x: event.clientX, y: event.clientY, size };
		(event.currentTarget as HTMLElement).setPointerCapture(event.pointerId);
	}

	function resize(event: PointerEvent): void {
		if (!isResizing) return;
		size = Math.max(48, Math.min(96, resizeStart.size + event.clientX - resizeStart.x));
	}

	function clampPosition(): void {
		position = {
			right: Math.max(8, Math.min(Math.max(8, window.innerWidth - size - 8), position.right)),
			bottom: Math.max(8, Math.min(Math.max(8, window.innerHeight - size - 8), position.bottom))
		};
	}

	onMount(() => {
		const saved = localStorage.getItem('quran-caption-assistant-position');
		if (saved) {
			try {
				const data = JSON.parse(saved);
				position = data.position ?? position;
				size = data.size ?? size;
			} catch { /* ignore invalid local preference */ }
		}
		resetIdle();
		window.addEventListener('resize', clampPosition);
		return () => {
			if (pressTimer) clearTimeout(pressTimer);
			if (idleTimer) clearTimeout(idleTimer);
			window.removeEventListener('resize', clampPosition);
			localStorage.setItem('quran-caption-assistant-position', JSON.stringify({ position, size }));
		};
	});
</script>

	<div class:idle={isIdle && !open} class:open-right={open && typeof window !== 'undefined' && position.right > window.innerWidth / 2} class="assistant-layer" style={`right:${position.right}px; bottom:${position.bottom}px;`}>
	{#if open}
		<section class="assistant-panel" style={`width:${Math.min(360, Math.max(280, size * 5))}px;`} aria-label="مساعد Quran Caption">
			<header>
				<div><strong>مساعد Quran Caption</strong><small>يفهم أوامر التطبيق ويطلب موافقتك</small></div>
				<button class="close" type="button" onclick={() => (open = false)} aria-label="إغلاق">×</button>
			</header>
			<div class="messages">
				{#each messages as message}
					<div class:mine={message.role === 'user'} class="message">{message.text}</div>
				{/each}
				{#if isTyping}<div class="typing" aria-label="المساعد يفكر"><i></i><i></i><i></i></div>{/if}
			</div>
			{#if pendingAction}
				<div class="approval"><span>عملية معلقة: {pendingAction.label}</span><div><button type="button" class="approve" onclick={confirmAction}>موافق وتنفيذ</button><button type="button" class="cancel" onclick={cancelAction}>إلغاء</button></div></div>
			{/if}
			<div class="quick-actions">
				<button type="button" onclick={() => { input = 'اعمل caption للصوت'; send(); }}>كابتشن الصوت</button>
				<button type="button" onclick={() => { input = 'افتح البحث'; send(); }}>البحث</button>
				<button type="button" onclick={() => { input = 'جهز التصدير'; send(); }}>التصدير</button>
			</div>
			<form onsubmit={(event) => { event.preventDefault(); send(); }}>
				<input bind:value={input} placeholder="اكتب طلبك…" aria-label="رسالة للمساعد" />
				<button class="send" type="submit">إرسال</button>
			</form>
		</section>
	{/if}
	<button class={`assistant-bubble status-${status}`} style={`width:${size}px;height:${size}px;`} type="button" aria-label="فتح مساعد Quran Caption" onpointerdown={startPointer} onpointermove={movePointer} onpointerup={endPointer} onpointercancel={endPointer}>
		<span class="bubble-icon">ق</span>
		<span class="bubble-label">{status === 'working' ? 'يعمل' : status === 'error' ? 'مشكلة' : 'مساعد'}</span>
		<span data-resize class="resize-handle" onpointerdown={startResize} onpointermove={resize} onpointerup={endPointer}></span>
	</button>
</div>

<style>
	.assistant-layer { position: fixed; z-index: 80; display: flex; flex-direction: column; align-items: flex-end; gap: 10px; direction: rtl; transition: opacity .35s ease, transform .35s ease, bottom .35s ease; }
	.assistant-layer.idle { opacity: .48; transform: translateY(8px); }
	.assistant-layer.idle:hover, .assistant-layer.idle:focus-within { opacity: 1; transform: translateY(0); }
	.assistant-bubble { position: relative; border: 2px solid rgba(255,255,255,.85); border-radius: 999px; color: white; box-shadow: 0 8px 24px rgba(0,0,0,.35); touch-action: none; user-select: none; cursor: pointer; transition: background .2s ease, box-shadow .2s ease; }
	.status-ready { background: linear-gradient(145deg,#1687d9,#07549c); }
	.status-working { background: linear-gradient(145deg,#d54a38,#8e1d1d); animation: pulse 1s infinite; }
	.status-success { background: linear-gradient(145deg,#27b86a,#08753d); }
	.status-error { background: linear-gradient(145deg,#e34b43,#901f2a); }
	@keyframes pulse { 50% { box-shadow: 0 0 0 8px rgba(213,74,56,.18), 0 8px 24px rgba(0,0,0,.35); } }
	.bubble-icon { display: block; font-size: 1.45rem; font-weight: 800; line-height: 1; }
	.bubble-label { display: block; font-size: .57rem; margin-top: 2px; }
	.resize-handle { position: absolute; left: 1px; bottom: 5px; width: 13px; height: 13px; border-left: 2px solid white; border-bottom: 2px solid white; border-radius: 0 0 0 8px; cursor: nwse-resize; }
	.assistant-layer.open-right { align-items: flex-start; }
	.assistant-panel { max-width: calc(100vw - 24px); max-height: min(560px, calc(100vh - 130px)); display: flex; flex-direction: column; overflow: hidden; color: #e7edf7; background: #172235; border: 1px solid #365171; border-radius: 18px; box-shadow: 0 18px 48px rgba(0,0,0,.45); }
	header { display:flex; justify-content:space-between; align-items:center; padding: 14px 16px; background:#203653; }
	header strong, header small { display:block; } header small { margin-top:3px; color:#a7bad2; font-size:.75rem; }
	.close { color:white; background:transparent; border:0; font-size:1.5rem; cursor:pointer; }
	.messages { padding:12px; overflow:auto; display:flex; flex-direction:column; gap:8px; }
	.message { align-self:flex-start; max-width:90%; padding:9px 11px; border-radius:12px 12px 12px 3px; background:#263b56; line-height:1.5; font-size:.86rem; }
	.message.mine { align-self:flex-end; background:#1267a6; border-radius:12px 12px 3px 12px; }
	.typing { align-self:flex-start; display:flex; gap:4px; padding:10px 12px; border-radius:12px; background:#263b56; }
	.typing i { width:6px; height:6px; border-radius:50%; background:#a7bad2; animation: typing-bounce 1s infinite ease-in-out; }
	.typing i:nth-child(2) { animation-delay:.15s; } .typing i:nth-child(3) { animation-delay:.3s; }
	@keyframes typing-bounce { 0%,60%,100% { transform:translateY(0); opacity:.45; } 30% { transform:translateY(-4px); opacity:1; } }
	.quick-actions { display:flex; gap:6px; padding:0 12px 10px; flex-wrap:wrap; }
	.quick-actions button, .send, .approval button { border:0; border-radius:999px; padding:7px 11px; background:#315579; color:white; cursor:pointer; }
	.approval { margin:0 12px 8px; padding:10px; border:1px solid #607c9f; border-radius:12px; background:#213852; font-size:.8rem; }
	.approval div { display:flex; gap:6px; margin-top:8px; }
	.approval .approve { background:#159957; } .approval .cancel { background:#8d3a46; }
	form { display:flex; gap:6px; padding:10px; border-top:1px solid #304765; }
	input { min-width:0; flex:1; border:1px solid #4b6684; border-radius:10px; padding:9px; color:white; background:#101a2a; outline:none; }
	.send { background:#1687d9; }
</style>
