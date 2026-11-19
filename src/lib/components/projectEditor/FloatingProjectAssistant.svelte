<script lang="ts">
	import { onMount } from 'svelte';
	import { globalState } from '$lib/runes/main.svelte';
	import { ProjectEditorTabs } from '$lib/classes';
	import RecitersManager from '$lib/classes/Reciter';

	type Message = { role: 'user' | 'assistant'; text: string };

	let open = $state(false);
	let input = $state('');
	let isDragging = $state(false);
	let isResizing = $state(false);
	let position = $state({ right: 24, bottom: 84 });
	let size = $state(58);
	let pressTimer: ReturnType<typeof setTimeout> | undefined;
	let dragStart = { x: 0, y: 0, right: 24, bottom: 84 };
	let resizeStart = { x: 0, y: 0, size: 58 };
	let messages = $state<Message[]>([
		{
			role: 'assistant',
			text: 'مرحبًا، أنا مساعد Quran Caption. أستطيع إرشادك داخل المشروع وفتح قسم الشيوخ أو الترجمة أو النمط أو التصدير.'
		}
	]);

	function setTab(tab: ProjectEditorTabs): void {
		if (!globalState.currentProject) return;
		globalState.currentProject.projectEditorState.currentTab = tab;
		open = false;
	}

	function answer(question: string): string {
		const q = question.trim().toLowerCase();
		if (!q) return 'اكتب طلبًا قصيرًا، مثل: اعرض الشيوخ، افتح الترجمة، أو كيف أصدّر الفيديو؟';
		if (q.includes('شيخ') || q.includes('قارئ') || q.includes('reciter')) {
			const reciters = RecitersManager.getRecitersWithCustomOnes();
			const names = reciters.slice(0, 12).map((r) => r.label).join('، ');
			return `يوجد ${reciters.length} شيخًا/قارئًا متاحًا حاليًا. من القائمة: ${names}${reciters.length > 12 ? '…' : ''}. لإضافة شيخ جديد نحتاج رابط الصوت أو ملفاته ومصدرًا موثوقًا، ولن أضيفه تلقائيًا بدون تأكيد.`;
		}
		if (q.includes('ترجم') || q.includes('translation')) {
			setTab(ProjectEditorTabs.Translations);
			return 'فتحت لك قسم الترجمات. اختر الترجمة من هناك، ثم راجعها قبل التصدير.';
		}
		if (q.includes('نمط') || q.includes('ستايل') || q.includes('style')) {
			setTab(ProjectEditorTabs.Style);
			return 'فتحت لك قسم النمط لتعديل شكل الآيات والترجمة.';
		}
		if (q.includes('صدّر') || q.includes('تصدير') || q.includes('export')) {
			setTab(ProjectEditorTabs.Export);
			return 'فتحت لك قسم التصدير. راجع المدى والإعدادات واضغط تصدير، وسأترك لك التأكيد النهائي حتى لا يبدأ تصدير طويل بالخطأ.';
		}
		if (q.includes('فيديو') || q.includes('video')) {
			setTab(ProjectEditorTabs.VideoEditor);
			return 'فتحت لك محرر الفيديو. ارفع الفيديو أو الصوت من مدير الوسائط، ثم انتقل إلى الترجمة أو النمط.';
		}
		if (q.includes('مساعدة') || q.includes('ماذا') || q.includes('help')) {
			return 'أستطيع حاليًا فتح محرر الفيديو، الترجمات، الأنماط، والتصدير، وعرض قائمة الشيوخ. العمليات التي تحذف ملفات أو تضيف مصادر خارجية ستحتاج تأكيدًا صريحًا.';
		}
		return 'فهمت الطلب جزئيًا. جرّب: «اعرض الشيوخ»، «افتح الترجمة»، «افتح النمط»، «افتح التصدير»، أو «افتح الفيديو».';
	}

	function send(): void {
		const text = input.trim();
		if (!text) return;
		messages = [...messages, { role: 'user', text }, { role: 'assistant', text: answer(text) }];
		input = '';
	}

	function startPointer(event: PointerEvent): void {
		if ((event.target as HTMLElement).closest('[data-resize]')) return;
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
	}

	function endPointer(): void {
		if (pressTimer) clearTimeout(pressTimer);
		if (!isDragging && !isResizing) open = !open;
		isDragging = false;
		isResizing = false;
	}

	function startResize(event: PointerEvent): void {
		event.stopPropagation();
		isResizing = true;
		resizeStart = { x: event.clientX, y: event.clientY, size };
		(event.currentTarget as HTMLElement).setPointerCapture(event.pointerId);
	}

	function resize(event: PointerEvent): void {
		if (!isResizing) return;
		size = Math.max(48, Math.min(96, resizeStart.size + event.clientX - resizeStart.x));
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
		return () => {
			localStorage.setItem('quran-caption-assistant-position', JSON.stringify({ position, size }));
		};
	});
</script>

<div class="assistant-layer" style={`right:${position.right}px; bottom:${position.bottom}px;`}>
	{#if open}
		<section class="assistant-panel" style={`width:${Math.min(360, Math.max(280, size * 5))}px;`} aria-label="مساعد Quran Caption">
			<header>
				<div><strong>مساعد Quran Caption</strong><small>يساعدك داخل المشروع</small></div>
				<button class="close" type="button" onclick={() => (open = false)} aria-label="إغلاق">×</button>
			</header>
			<div class="messages">
				{#each messages as message}
					<div class:mine={message.role === 'user'} class="message">{message.text}</div>
				{/each}
			</div>
			<div class="quick-actions">
				<button type="button" onclick={() => { input = 'اعرض الشيوخ'; send(); }}>الشيوخ</button>
				<button type="button" onclick={() => { input = 'افتح الترجمة'; send(); }}>الترجمة</button>
				<button type="button" onclick={() => { input = 'افتح التصدير'; send(); }}>التصدير</button>
			</div>
			<form onsubmit={(event) => { event.preventDefault(); send(); }}>
				<input bind:value={input} placeholder="اكتب طلبك…" aria-label="رسالة للمساعد" />
				<button class="send" type="submit">إرسال</button>
			</form>
		</section>
	{/if}
	<button
		class="assistant-bubble"
		style={`width:${size}px;height:${size}px;`}
		type="button"
		aria-label="فتح مساعد Quran Caption"
		onpointerdown={startPointer}
		onpointermove={movePointer}
		onpointerup={endPointer}
		onpointercancel={endPointer}
	>
		<span class="bubble-icon">ق</span>
		<span class="bubble-label">مساعد</span>
		<span data-resize class="resize-handle" onpointerdown={startResize} onpointermove={resize} onpointerup={endPointer}></span>
	</button>
</div>

<style>
	.assistant-layer { position: fixed; z-index: 80; display: flex; flex-direction: column; align-items: flex-end; gap: 10px; direction: rtl; }
	.assistant-bubble { position: relative; border: 2px solid rgba(255,255,255,.8); border-radius: 999px; background: linear-gradient(145deg,#1687d9,#07549c); color: white; box-shadow: 0 8px 24px rgba(0,0,0,.35); touch-action: none; user-select: none; cursor: pointer; }
	.bubble-icon { display: block; font-size: 1.45rem; font-weight: 800; line-height: 1; }
	.bubble-label { display: block; font-size: .57rem; margin-top: 2px; }
	.resize-handle { position: absolute; left: 1px; bottom: 5px; width: 13px; height: 13px; border-left: 2px solid white; border-bottom: 2px solid white; border-radius: 0 0 0 8px; cursor: nwse-resize; }
	.assistant-panel { max-width: calc(100vw - 24px); max-height: min(560px, calc(100vh - 130px)); display: flex; flex-direction: column; overflow: hidden; color: #e7edf7; background: #172235; border: 1px solid #365171; border-radius: 18px; box-shadow: 0 18px 48px rgba(0,0,0,.45); }
	header { display:flex; justify-content:space-between; align-items:center; padding: 14px 16px; background:#203653; }
	header strong, header small { display:block; } header small { margin-top:3px; color:#a7bad2; font-size:.75rem; }
	.close { color:white; background:transparent; border:0; font-size:1.5rem; cursor:pointer; }
	.messages { padding:12px; overflow:auto; display:flex; flex-direction:column; gap:8px; }
	.message { align-self:flex-start; max-width:90%; padding:9px 11px; border-radius:12px 12px 12px 3px; background:#263b56; line-height:1.5; font-size:.86rem; }
	.message.mine { align-self:flex-end; background:#1267a6; border-radius:12px 12px 3px 12px; }
	.quick-actions { display:flex; gap:6px; padding:0 12px 10px; flex-wrap:wrap; }
	.quick-actions button, .send { border:0; border-radius:999px; padding:7px 11px; background:#315579; color:white; cursor:pointer; }
	form { display:flex; gap:6px; padding:10px; border-top:1px solid #304765; }
	input { min-width:0; flex:1; border:1px solid #4b6684; border-radius:10px; padding:9px; color:white; background:#101a2a; outline:none; }
	.send { background:#1687d9; }
</style>
