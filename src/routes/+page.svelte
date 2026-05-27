<script lang="ts">
	import { onMount, onDestroy, untrack } from 'svelte';
	import { fade, fly, scale } from 'svelte/transition';

	// ==========================================
	// SVELTE 5 STATE VARIABLES
	// ==========================================
	let gameMode = $state<'solo' | 'vs'>('vs');
	let gameState = $state<'lobby' | 'ready' | 'listening' | 'success' | 'failure' | 'gameover'>('lobby');
	let currentPlayer = $state<1 | 2>(1);
	
	// Player Stats
	let player1Score = $state(0);
	let player2Score = $state(0);
	let player1Lives = $state(3);
	let player2Lives = $state(3);
	let soloHighScore = $state(0);
	let roundsCount = $state(1);

	// Shiritori Chain State
	let lastWord = $state('しりとり');
	let lastWordReading = $state('しりとり');
	let usedWords = $state<string[]>(['しりとり']); // List of already shouted words

	// Web Audio Logarithmic dBFS States (Decibel range from -60 dB to +10 dB)
	let targetVolume = $state(-45.0); // Starting threshold in decibels
	let currentVolume = $state(-60.0);   // Real-time decibels
	let peakVolume = $state(-60.0);      // Max decibels recorded in current speech window
	let smoothedVolume = $state(-60.0);  // Damped decibels for the visual gauge
	let micSensitivity = $state(2.5); // Gain multiplier
	let ambientNoiseLevel = $state(-55.0); // Calibrated decibel sound floor

	// Speech Recognition States
	let recognizedText = $state('');  
	let inputReading = $state('');    
	let isListening = $state(false);
	let activeSpeechDuration = $state(0); // Cumulative seconds of physical vocal activity
	
	// Calibration States
	let isCalibrating = $state(false);
	let calibrationCountdown = $state(3);

	// UI & Rock Special Effects
	let isShaking = $state(false);
	let isFlashing = $state(false); // Live house strobe overlay
	let isValidatingWord = $state(false); // Wikipedia dictionary lookup fader state
	let showHelpModal = $state(false);
	let speechRecognitionSupported = $state(true);
	let micPermission = $state<'prompt' | 'granted' | 'denied'>('prompt');
	let errorMessage = $state('');

	// Plain Non-Reactive Mutable JS Variables (Decoupled from Svelte 5 tracking)
	let stream: MediaStream | null = null;
	let audioContext: AudioContext | null = null;
	let analyserNode: AnalyserNode | null = null;
	let speechRecognition: any = null;
	let canvasElement = $state<HTMLCanvasElement | null>(null);

	// Particle System for Visualizer
	interface Particle {
		x: number;
		y: number;
		vx: number;
		vy: number;
		radius: number;
		color: string;
		alpha: number;
	}
	let particles: Particle[] = [];

	// ==========================================
	// DECIBEL CONVERSION HELPERS
	// ==========================================

	// Map decibel range [-60 dB, +10 dB] linearly to [0, 1] for visual elements (70 dB dynamic range)
	function dbToFraction(db: number): number {
		if (db <= -60.0) return 0;
		return Math.min(1.0, (db + 60.0) / 70.0);
	}

	// Map decibel range [-60 dB, +10 dB] to percentage [0, 100] for text display
	function dbToPercent(db: number): number {
		return dbToFraction(db) * 100;
	}

	// ==========================================
	// JAPANESE NORMALIZATION & SHIRITORI RULES
	// ==========================================
	const SHIRITORI_DICT: Record<string, string> = {
		// り
		'りんご': 'りんご', '林檎': 'りんご', 'りす': 'りす', '栗鼠': 'りす', 'らっぱ': 'らっぱ', 'ラッパ': 'らっぱ', 'らくだ': 'らくだ', '駱駝': 'らくだ',
		// ご
		'ごりら': 'ごりら', 'ゴリラ': 'ごりら', 'ごま': 'ごま', '胡麻': 'ごま', 'ごはん': 'ごはん', '御飯': 'ごはん',
		// ら
		'らいおん': 'らいおん', 'ライオン': 'らいおん', 'らーめん': 'らーめん', 'ラーメン': 'らーめん', 'らじお': 'らじお', 'ラジオ': 'らじお',
		// す
		'すいか': 'すいか', '西瓜': 'すいか', 'すずめ': 'すずめ', '雀': 'すずめ', 'すし': 'すし', '寿司': 'すし',
		// か
		'からす': 'からす', '烏': 'からす', 'かめ': 'かめ', '亀': 'かめ', 'かえる': 'かえる', '蛙': 'かえる', 'かさ': 'かさ', '傘': 'かさ',
		// め
		'めだか': 'めだか', '目高': 'めだか', 'めがね': 'めがね', '眼鏡': 'めがね', 'めろん': 'めろん', 'メロン': 'めろん',
		// ね
		'ねこ': 'ねこ', '猫': 'ねこ', 'ねずみ': 'ねずみ', '鼠': 'ねずみ', 'ねんど': 'ねんど', '粘土': 'ねんど',
		// こ
		'こあら': 'こあら', 'コアラ': 'こあら', 'こま': 'こま', '独楽': 'こま', 'こころ': 'こころ', '心': 'こころ',
		// た
		'たいこ': 'たいこ', '太鼓': 'たいこ', 'たぬき': 'たぬき', '狸': 'たぬき', 'たまご': 'たまご', '卵': 'たまご',
		// ま
		'まくら': 'まくら', '枕': 'まくら', 'まつり': 'まつり', '祭り': 'まつり', 'まめ': 'まめ', '豆': 'まめ',
		// つ
		'つくえ': 'つくえ', '机': 'つくえ', 'つばめ': 'つばめ', '燕': 'つばめ', 'つみき': 'つみき', '積木': 'つみき',
		// え
		'えんぴつ': 'えんぴつ', '鉛筆': 'えんぴつ', 'えほん': 'えほん', '絵本': 'えほん', 'えき': 'えき', '駅': 'えき',
		// き
		'きつね': 'きつね', '狐': 'きつね', 'きりん': 'きりん', 'キリン': 'きりん', 'きく': 'きく', '菊': 'きく',
		// ぬ
		'ぬりえ': 'ぬりえ', '塗り絵': 'ぬりえ', 'ぬいぐるみ': 'ぬいぐるみ', '縫いぐるみ': 'ぬいぐるみ',
		// は
		'はむすたー': 'はむすたー', 'ハムスター': 'はむすたー', 'はな': 'はな', '花': 'はな', 'はさみ': 'はさみ', '鋏': 'はさみ',
		// み
		'みかん': 'みかん', '蜜柑': 'みかん', 'みず': 'みず', '水': 'みず', 'みみ': 'みみ', '耳': 'みみ',
		// い
		'いぬ': 'いぬ', '犬': 'いぬ', 'いちご': 'いちご', '苺': 'いちご', 'いえ': 'いえ', '家': 'いえ',
		// う
		'うさぎ': 'うさぎ', '兎': 'うさぎ', 'うちわ': 'うちわ', '団扇': 'うちわ', 'うみ': 'うみ', '海': 'うみ'
	};

	// Convert Katakana to Hiragana
	function katakanaToHiragana(src: string): string {
		return src.replace(/[\u30a1-\u30f6]/g, (match) => {
			return String.fromCharCode(match.charCodeAt(0) - 0x60);
		});
	}

	// Clean Hiragana text
	function cleanHiragana(src: string): string {
		let text = src.trim();
		text = katakanaToHiragana(text);
		return text;
	}

	// Extract the starting character of a word
	function getStartChar(word: string): string {
		const clean = cleanHiragana(word);
		return clean ? clean.charAt(0) : '';
	}

	// Extract the ending character(s) of a word, applying Shiritori rules
	function getEndChar(word: string): string {
		let clean = cleanHiragana(word);
		if (!clean) return '';

		// Remove trailing long vowel markers (ー)
		while (clean.endsWith('ー') && clean.length > 1) {
			clean = clean.slice(0, -1);
		}

		if (clean.length === 0) return '';

		const lastChar = clean.charAt(clean.length - 1);

		// List of small characters that should be combined with the preceding character
		const smallChars = ['ぁ', 'ぃ', 'ぅ', 'ぇ', 'ぉ', 'ゃ', 'ゅ', 'ょ', 'ゎ'];

		if (smallChars.includes(lastChar) && clean.length > 1) {
			// Return the last two characters combined (e.g., "しゅ")
			return clean.slice(-2);
		}

		// Map other small characters like 'っ' to standard sizes
		const smallToLarge: Record<string, string> = {
			'っ': 'つ', 'ヶ': 'け', 'ヵ': 'か'
		};

		return smallToLarge[lastChar] || lastChar;
	}

	// Resolve the Hiragana reading of a word
	function resolveReading(word: string): string {
		const cleaned = word.trim();
		if (SHIRITORI_DICT[cleaned]) {
			return SHIRITORI_DICT[cleaned];
		}
		
		const hira = cleanHiragana(cleaned);
		const hiraganaRegex = /^[\u3040-\u309Fー\s]+$/;
		if (hiraganaRegex.test(hira)) {
			return hira;
		}

		return ''; 
	}

	// ==========================================
	// AUDIO & SPEECH ENGINES INITIALIZATION
	// ==========================================

	// Request mic access and setup AudioContext
	async function initAudio() {
		try {
			stream = await navigator.mediaDevices.getUserMedia({
				audio: {
					echoCancellation: false,
					noiseSuppression: false,
					autoGainControl: false
				}
			});
			micPermission = 'granted';

			const AudioContextClass = window.AudioContext || (window as any).webkitAudioContext;
			audioContext = new AudioContextClass();
			analyserNode = audioContext.createAnalyser();
			analyserNode.fftSize = 512;

			const source = audioContext.createMediaStreamSource(stream);
			source.connect(analyserNode);

			errorMessage = '';
			return true;
		} catch (err: any) {
			console.error('Error accessing microphone:', err);
			micPermission = 'denied';
			errorMessage = 'マイクの使用が許可されていないか、マイクが見つかりません。設定からマイクの利用を許可してください。';
			return false;
		}
	}

	// Start Speech Recognition
	function startListening() {
		if (typeof window === 'undefined') return;

		// Clean states
		recognizedText = '';
		inputReading = '';
		peakVolume = -60.0;
		activeSpeechDuration = 0;
		errorMessage = '';

		if (!speechRecognition && speechRecognitionSupported) {
			const SpeechRecognitionClass = (window as any).SpeechRecognition || (window as any).webkitSpeechRecognition;
			if (!SpeechRecognitionClass) {
				speechRecognitionSupported = false;
				errorMessage = '音声認識非対応のブラウザです。マイクでの音量計測は可能ですが、単語は手動で入力してください。';
				
				isListening = true;
				gameState = 'listening';
				return;
			}

			speechRecognition = new SpeechRecognitionClass();
			speechRecognition.lang = 'ja-JP';
			speechRecognition.continuous = true; // Keep active while holding spacebar!
			speechRecognition.interimResults = false; // PREVENT CHREAT PREDICTIVE RESULTS!

			speechRecognition.onstart = () => {
				isListening = true;
				gameState = 'listening';
			};

			speechRecognition.onresult = (event: any) => {
				// Combine all recognized transcript segments during the spacebar hold session
				let transcript = '';
				for (let i = 0; i < event.results.length; i++) {
					transcript += event.results[i][0].transcript;
				}
				recognizedText = transcript;
				inputReading = cleanHiragana(transcript);
			};

			speechRecognition.onerror = (event: any) => {
				console.error('Speech recognition error:', event.error);
				if (event.error === 'no-speech') {
					return;
				}
				errorMessage = '音声認識エラー: ' + event.error;
				isListening = false;
			};

			speechRecognition.onend = () => {
				const stateVal = untrack(() => gameState);
				const recognizedVal = untrack(() => recognizedText);

				if (stateVal === 'listening') {
					isListening = false; // Safely stop Svelte listening loop
					
					if (!recognizedVal.trim()) {
						// Quietly return to ready if no speech was captured
						gameState = 'ready';
					} else {
						processTurnImmediately();
					}
				}
			};
		}

		if (!speechRecognitionSupported) {
			isListening = true;
			gameState = 'listening';
			setTimeout(() => {
				const stateVal = untrack(() => gameState);
				if (stateVal === 'listening') {
					stopListening();
				}
			}, 4500);
			return;
		}

		try {
			speechRecognition.start();
		} catch (err) {
			isListening = true;
			gameState = 'listening';
		}
	}

	// Stop Listening manually
	function stopListening() {
		// Ask Speech Engine to stop capturing. All turn evaluations are processed 
		// inside the onend handler once all final transcriptions complete asynchronously.
		if (speechRecognition && speechRecognitionSupported) {
			try {
				speechRecognition.stop();
			} catch (e) {}
		}
	}

	// Check if a word exists in Wikipedia and guess its Hiragana reading via intro extract & title
	async function checkWordAndGetReading(word: string): Promise<{ exists: boolean; guessedReading: string }> {
		try {
			const cleaned = word.replace(/[。、.!?？]/g, '').trim();
			if (!cleaned) return { exists: false, guessedReading: '' };

			// Request intro extract in plain text and follow redirects
			const url = `https://ja.wikipedia.org/w/api.php?action=query&format=json&origin=*&prop=extracts&exintro=1&explaintext=1&redirects=1&titles=${encodeURIComponent(cleaned)}`;
			const response = await fetch(url);
			const data = await response.json();
			
			if (data && data.query && data.query.pages) {
				const pages = data.query.pages;
				const pageId = Object.keys(pages)[0];
				
				if (pageId === '-1') {
					return { exists: false, guessedReading: '' };
				}
				
				const page = pages[pageId];
				const title = page.title || '';
				const extract = page.extract || '';
				
				let guessedReading = '';
				const hiraganaRegex = /^[\u3040-\u309Fー\s]+$/;

				// 1. If title itself is Katakana/Hiragana, use it
				const cleanTitle = cleanHiragana(title);
				if (hiraganaRegex.test(cleanTitle)) {
					guessedReading = cleanTitle;
				} else {
					// 2. Parse brackets in first sentence (e.g., "リンゴ（林檎、学名：...）") to extract reading
					const match = extract.match(/(?:^[^（(]*)[（(]([^）)]+)[）)]/);
					if (match && match[1]) {
						const bracketContent = match[1];
						const parts = bracketContent.split(/[、,;；・]/);
						for (const part of parts) {
							const cleanPart = cleanHiragana(part.trim());
							if (hiraganaRegex.test(cleanPart) && cleanPart.length > 0) {
								guessedReading = cleanPart;
								break;
							}
						}
					}
				}

				// Fallback to normalized word itself if no bracket reading could be matched
				if (!guessedReading) {
					guessedReading = cleanHiragana(cleaned);
				}

				return { exists: true, guessedReading };
			}
			return { exists: false, guessedReading: '' };
		} catch (err) {
			console.error('Wikipedia Search API failed:', err);
			return { exists: true, guessedReading: cleanHiragana(word) }; // Graceful fallback
		}
	}

	// Process and evaluate turn immediately
	async function processTurnImmediately() {
		const rawWord = recognizedText.trim();

		// Silent check - if empty on key release, return to ready so player can press spacebar again
		if (!rawWord) {
			gameState = 'ready';
			return;
		}

		// 1. Predictive Auto-Correct (Cheat) Detector Heuristic
		// Reject turn if vocal duration is suspiciously short relative to word length,
		// suggesting the browser's Speech API auto-completed the word.
		const charCount = rawWord.length;
		let isCheat = false;

		// If the user shouts for less than 0.5 seconds but gets a 4+ character word, it's a cheat.
		if (charCount >= 4 && activeSpeechDuration < 0.5) {
			isCheat = true;
		} else {
			// General heuristic: min 0.12s per character, with a 0.3s baseline gate.
			const minRequiredDuration = Math.max(0.3, charCount * 0.12);
			if (charCount >= 3 && activeSpeechDuration < minRequiredDuration) {
				isCheat = true;
			}
		}

		if (isCheat) {
			triggerShake();
			deductLife();
			triggerFailure('もっとはっきり発音してください');
			return;
		}

		// 2. Lock UI and trigger strict Wikipedia validation & reading guesser
		isValidatingWord = true;
		const { exists, guessedReading } = await checkWordAndGetReading(rawWord);
		isValidatingWord = false;

		if (!exists) {
			triggerShake();
			deductLife();
			triggerFailure(`そんな言葉はメタル辞書にない！ (Wikipedia未登録: 「${rawWord}」)`);
			return;
		}

		// Resolve reading of spoken word
		let reading = resolveReading(rawWord) || guessedReading;
		let linkBypassed = false;

		// If still no valid hiragana reading is resolved, bypass the link check
		const hiraganaRegex = /^[\u3040-\u309Fー\s]+$/;
		if (!reading || !hiraganaRegex.test(reading)) {
			linkBypassed = true;
			reading = cleanHiragana(rawWord); 
		}

		// 3. Repeated word checker (Once used, word becomes invalid)
		const cleanedRaw = cleanHiragana(rawWord);
		if (usedWords.includes(reading) || usedWords.includes(cleanedRaw)) {
			triggerShake();
			deductLife();
			triggerFailure(`その言葉はすでに使われています！ (「${rawWord}」)`);
			return;
		}

		// 3. "ん" Checker 
		if (!linkBypassed) {
			const endChar = getEndChar(reading);
			if (endChar === 'ん') {
				deductLife();
				triggerFailure('「ん」がついたため、ゲームオーバー！');
				return;
			}
		}

		// 4. Shiritori link check
		if (!linkBypassed && lastWordReading) {
			const expectedStart = getEndChar(lastWordReading);
			const cleanReading = cleanHiragana(reading);
			const actualStart = cleanReading.slice(0, expectedStart.length);

			if (expectedStart !== actualStart) {
				triggerShake();
				deductLife();
				triggerFailure(`しりとり不成立！「${expectedStart}」から始まる単語ではありません！ (認識: ${rawWord})`);
				return;
			}
		}

		// 5. Logarithmic Decibel Check (Direct comparison on -60 to +10 dBFS scale)
		if (peakVolume < targetVolume) {
			triggerShake();
			deductLife();
			triggerFailure(`声量が足りません！(目標: ${targetVolume.toFixed(0)} dB / 計測: ${peakVolume.toFixed(1)} dB)`);
			return;
		}

		// TURN SUCCESS!
		lastWord = rawWord;
		usedWords = [...usedWords, reading]; // Register in repeated words list

		// Carry link forward
		if (linkBypassed) {
			const last = reading.charAt(reading.length - 1);
			const hiraganaRegex = /^[\u3040-\u309F]$/;
			if (hiraganaRegex.test(last) && last !== 'ん') {
				lastWordReading = reading;
			} else {
				lastWordReading = lastWordReading;
			}
		} else {
			lastWordReading = reading;
		}

		// TARGET VOLUME HANDOVER FIX:
		// Direct handover! The peak volume from this success becomes the exact target decibel for the next turn.
		targetVolume = peakVolume;

		if (gameMode === 'solo') {
			player1Score++;
			if (player1Score > soloHighScore) {
				soloHighScore = player1Score;
			}
		} else {
			if (currentPlayer === 1) {
				player1Score++;
			} else {
				player2Score++;
			}
		}

		triggerSuccess();
	}

	// Trigger shake animation
	function triggerShake() {
		isShaking = true;
		setTimeout(() => {
			isShaking = false;
		}, 400);
	}

	// Deduct a life
	function deductLife() {
		if (gameMode === 'solo') {
			player1Lives--;
		} else {
			if (currentPlayer === 1) {
				player1Lives--;
			} else {
				player2Lives--;
			}
		}
	}

	// Success screen transition (Includes Rock Strobe Flash)
	function triggerSuccess() {
		isFlashing = true;
		gameState = 'success';
		setTimeout(() => {
			isFlashing = false;
		}, 400);

		setTimeout(() => {
			if (player1Lives <= 0 || (gameMode === 'vs' && player2Lives <= 0)) {
				gameState = 'gameover';
			} else {
				if (gameMode === 'vs') {
					currentPlayer = currentPlayer === 1 ? 2 : 1;
				}
				roundsCount++;
				gameState = 'ready';
			}
		}, 2000);
	}

	// Failure screen transition
	function triggerFailure(msg: string) {
		errorMessage = msg;
		gameState = 'failure';
		setTimeout(() => {
			if (player1Lives <= 0 || (gameMode === 'vs' && player2Lives <= 0)) {
				gameState = 'gameover';
			} else {
				// Reduce target volume by 5 dB on failure to make it easier, capped at minimum -50 dB
				targetVolume = Math.max(-50.0, targetVolume - 5.0);
				if (gameMode === 'vs') {
					currentPlayer = currentPlayer === 1 ? 2 : 1;
				}
				gameState = 'ready';
			}
		}, 3500);
	}

	// Calibrate ambient noise in dBFS
	async function calibrateNoise() {
		if (!audioContext) {
			const active = await initAudio();
			if (!active) return;
		}

		isCalibrating = true;
		calibrationCountdown = 3;
		let noiseSamples: number[] = [];

		const interval = setInterval(() => {
			calibrationCountdown--;
			if (calibrationCountdown <= 0) {
				clearInterval(interval);
				
				if (noiseSamples.length > 0) {
					const avgNoise = noiseSamples.reduce((a, b) => a + b, 0) / noiseSamples.length;
					ambientNoiseLevel = Math.max(-60.0, Math.min(-20.0, avgNoise));
					// Target baseline calibrated to Ambient + 10 dB, clamped between -50 dB and -10 dB
					targetVolume = Math.max(-50.0, Math.min(-10.0, ambientNoiseLevel + 10.0));
				}
				isCalibrating = false;
			}
		}, 1000);

		const sampleLoop = () => {
			if (!isCalibrating) return;
			if (analyserNode) {
				const bufferLength = analyserNode.fftSize;
				const dataArray = new Float32Array(bufferLength);
				analyserNode.getFloatTimeDomainData(dataArray);
				
				let sum = 0;
				for (let i = 0; i < bufferLength; i++) {
					sum += dataArray[i] * dataArray[i];
				}
				const rms = Math.sqrt(sum / bufferLength);
				let db = -60.0;
				if (rms > 0.0001) {
					db = 20 * Math.log10(rms);
				}
				db = Math.max(-60.0, db);
				noiseSamples.push(db);
			}
			requestAnimationFrame(sampleLoop);
		};
		sampleLoop();
	}

	// Coward's Reset (Resets decibels to baseline)
	function cowardsReset() {
		if (gameState === 'ready' || gameState === 'listening') {
			deductLife();
			targetVolume = Math.max(-50.0, ambientNoiseLevel + 8.0);
			triggerShake();
			
			if (player1Lives <= 0 || (gameMode === 'vs' && player2Lives <= 0)) {
				gameState = 'gameover';
			} else {
				recognizedText = '';
				inputReading = '';
				peakVolume = -60.0;
				activeSpeechDuration = 0;
				gameState = 'ready';
			}
		}
	}

	// Start game
	async function startGame(mode: 'solo' | 'vs') {
		const success = await initAudio();
		if (!success) return;

		gameMode = mode;
		currentPlayer = 1;
		player1Score = 0;
		player2Score = 0;
		player1Lives = 3;
		player2Lives = 3;
		roundsCount = 1;
		
		lastWord = 'しりとり';
		lastWordReading = 'しりとり';
		usedWords = ['しりとり'];
		
		targetVolume = Math.max(-45.0, ambientNoiseLevel + 10.0);
		peakVolume = -60.0;
		currentVolume = -60.0;
		smoothedVolume = -60.0;
		activeSpeechDuration = 0;
		
		gameState = 'ready';
	}

	// Back to Lobby
	function returnToLobby() {
		gameState = 'lobby';
		errorMessage = '';
	}

	// ==========================================
	// HIGH-PERFORMANCE AUDIO CANVAS EFFECT (onMount)
	// ==========================================
	function spawnParticles(x: number, y: number, volumeFraction: number, pType: 1 | 2) {
		const count = Math.floor(volumeFraction * 20); // More particles for heavy theme!
		const color = '239, 68, 68'; // Burning Red for all!
		for (let i = 0; i < count; i++) {
			particles.push({
				x: x,
				y: y,
				vx: (Math.random() - 0.5) * 8,
				vy: -(Math.random() * 6 + volumeFraction * 12),
				radius: Math.random() * 5 + 1.5,
				color: color,
				alpha: 1.0
			});
		}
	}

	function updateParticles(width: number, height: number) {
		for (let i = particles.length - 1; i >= 0; i--) {
			const p = particles[i];
			p.x += p.vx;
			p.y += p.vy;
			p.vy += 0.12; // Higher gravity for volcanic feel
			p.alpha -= 0.02;

			if (p.alpha <= 0 || p.x < 0 || p.x > width || p.y > height) {
				particles.splice(i, 1);
			}
		}
	}

	// Decouple Canvas & Web Audio RMS parsing completely from Svelte 5 `$effect`
	onMount(() => {
		let animationFrameId: number;
		let lastFrameTime = Date.now();

		const render = () => {
			animationFrameId = requestAnimationFrame(render);
			
			if (!canvasElement) return;
			const ctx = canvasElement.getContext('2d');
			if (!ctx) return;

			const width = canvasElement.width;
			const height = canvasElement.height;

			// Extreme pitch-black canvas trace
			ctx.fillStyle = 'rgba(10, 10, 10, 0.25)';
			ctx.fillRect(0, 0, width, height);

			// Delta time calculation
			const now = Date.now();
			const deltaSeconds = (now - lastFrameTime) / 1000;
			lastFrameTime = now;

			const isListeningVal = untrack(() => isListening);
			const sensitivityVal = untrack(() => micSensitivity);
			const playerVal = untrack(() => currentPlayer);

			if (analyserNode && isListeningVal) {
				const bufferLength = analyserNode.fftSize;
				const dataArray = new Float32Array(bufferLength);
				analyserNode.getFloatTimeDomainData(dataArray);

				// Compute Root Mean Square (RMS) of dataArray scaled by gain multiplier
				let sum = 0;
				for (let i = 0; i < bufferLength; i++) {
					const val = dataArray[i] * sensitivityVal;
					sum += val * val;
				}
				const rms = Math.sqrt(sum / bufferLength);

				// Convert logarithmic RMS to decibels (dBFS) on expanded range [-60 dB to +10 dB]
				let db = -60.0;
				if (rms > 0.0001) {
					db = 20 * Math.log10(rms);
				}
				db = Math.max(-60.0, db);
				
				untrack(() => {
					currentVolume = db;
					if (db > peakVolume) {
						peakVolume = db;
					}
					smoothedVolume = smoothedVolume * 0.70 + db * 0.30;

					// Accumulate physical vocal duration (above gate limit -45 dB, adjusted for calibrated noise floor)
					const voiceThreshold = Math.max(-45.0, ambientNoiseLevel + 5.0);
					if (db > voiceThreshold) {
						activeSpeechDuration += deltaSeconds;
					}
				});

				const dbFract = dbToFraction(db);

				// Draw Burning Neon Red Wave (Heavy Metal Style)
				ctx.lineWidth = 4.0;
				ctx.strokeStyle = '#ef4444'; // Solid Danger Red
				ctx.shadowBlur = 24;
				ctx.shadowColor = 'rgba(239, 68, 68, 0.9)';
				ctx.beginPath();

				const sliceWidth = width / bufferLength;
				let x = 0;

				for (let i = 0; i < bufferLength; i++) {
					const v = dataArray[i] * sensitivityVal;
					const y = (v * height * 0.48) + (height / 2);

					if (i === 0) {
						ctx.moveTo(x, y);
					} else {
						ctx.lineTo(x, y);
					}

					// Spark particles trigger when volume is notable
					if (db > -35.0 && Math.random() < 0.3) {
						spawnParticles(x, y, dbFract, playerVal);
					}

					x += sliceWidth;
				}

				ctx.lineTo(width, height / 2);
				ctx.stroke();
				ctx.shadowBlur = 0;

				// Bottom volcanic lava glow fill
				ctx.fillStyle = 'rgba(239, 68, 68, 0.06)';
				ctx.beginPath();
				x = 0;
				ctx.moveTo(0, height);
				for (let i = 0; i < bufferLength; i++) {
					const v = dataArray[i] * sensitivityVal;
					const y = (v * height * 0.48) + (height / 2);
					ctx.lineTo(x, y);
					x += sliceWidth;
				}
				ctx.lineTo(width, height);
				ctx.closePath();
				ctx.fill();

			} else {
				// Idle background wave decaying toward silence (-60 dB)
				untrack(() => {
					smoothedVolume = smoothedVolume * 0.85 + (-60.0) * 0.15;
					currentVolume = -60.0;
				});

				ctx.lineWidth = 2.0;
				ctx.strokeStyle = 'rgba(239, 68, 68, 0.12)';
				ctx.shadowBlur = 0;
				ctx.beginPath();

				const time = Date.now() * 0.005;
				ctx.moveTo(0, height / 2);
				for (let x = 0; x < width; x++) {
					const y = Math.sin(x * 0.03 + time) * 3 + Math.cos(x * 0.02 - time) * 1 + (height / 2);
					ctx.lineTo(x, y);
				}
				ctx.stroke();
			}

			// Render active volcanic embers
			updateParticles(width, height);
			for (const p of particles) {
				ctx.fillStyle = `rgba(${p.color}, ${p.alpha})`;
				ctx.shadowBlur = p.alpha * 12;
				ctx.shadowColor = `rgba(${p.color}, ${p.alpha})`;
				ctx.beginPath();
				ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
				ctx.fill();
			}
			ctx.shadowBlur = 0;
		};

		// Start loop
		render();

		return () => {
			cancelAnimationFrame(animationFrameId);
		};
	});

	// Cleanup browser references on component unmount
	onDestroy(() => {
		if (stream) {
			stream.getTracks().forEach(track => track.stop());
		}
		if (audioContext) {
			audioContext.close();
		}
	});

	let isSpaceKeyPressed = false; // Prevent keyboard auto-repeat

	// KeyDown handlers (Push-to-Talk)
	function handleKeyDown(e: KeyboardEvent) {
		if (e.code === 'Space') {
			e.preventDefault();
			if (isSpaceKeyPressed) return; // Guard auto-repeat

			const stateVal = untrack(() => gameState);
			if (stateVal === 'ready') {
				isSpaceKeyPressed = true;
				startListening();
			}
		}
	}

	// KeyUp handlers (Push-to-Talk)
	function handleKeyUp(e: KeyboardEvent) {
		if (e.code === 'Space') {
			e.preventDefault();
			isSpaceKeyPressed = false;
			const stateVal = untrack(() => gameState);
			if (stateVal === 'listening') {
				stopListening();
			}
		}
	}
</script>

<svelte:window onkeydown={handleKeyDown} onkeyup={handleKeyUp} />

<!-- FULL SCREEN STROBE FLASH OVERLAY FOR CONCERT IMPACT -->
{#if isFlashing}
	<div out:fade={{ duration: 350 }} class="fixed inset-0 bg-red-600/90 z-50 pointer-events-none mix-blend-screen"></div>
{/if}

<!-- MAIN WRAPPER: INDUSTRIAL FLIGHT-CASE GRID -->
<main class="min-h-screen relative flex flex-col justify-between py-6 px-4 md:px-8 select-none transition-transform duration-100 bg-neutral-950 text-neutral-100 {isShaking ? 'animate-shake' : ''}">
	
	<!-- Cyberpunk Industrial Iron Mesh Backdrop Overlay -->
	<div class="absolute inset-0 bg-[radial-gradient(ellipse_at_center,rgba(239,68,68,0.03),rgba(10,10,10,1))] pointer-events-none z-0"></div>
	<div class="absolute inset-0 bg-[linear-gradient(rgba(255,255,255,0.003)_1px,transparent_1px),linear-gradient(90deg,rgba(255,255,255,0.003)_1px,transparent_1px)] bg-[size:24px_24px] pointer-events-none z-0"></div>

	<!-- ==========================================
	  HEADER SECTION
	========================================== -->
	<header class="w-full flex justify-between items-center max-w-5xl mx-auto relative z-10 border-b-4 border-neutral-900 pb-3">
		<button onclick={returnToLobby} class="flex items-center gap-2 group font-sans text-sm text-neutral-400 hover:text-red-500 transition-colors bg-transparent border-0 cursor-pointer">
			<span class="text-xl group-hover:scale-110 transition-transform">🔥</span> 
			<span class="uppercase tracking-widest font-black font-digital text-lg">HELL VOLUME SHIRITORI</span>
		</button>
		
		<div class="flex gap-4">
			<button 
				onclick={() => showHelpModal = true} 
				class="px-4 py-1.5 rounded-none bg-neutral-900 text-xs font-black tracking-widest text-neutral-300 hover:text-yellow-400 hover:bg-neutral-800 border-2 border-neutral-800 transition-all flex items-center gap-1.5 cursor-pointer uppercase"
			>
				📖 RULES
			</button>
		</div>
	</header>

	<!-- ==========================================
	  LOBBY SCREEN (HELL / METAL ARENA)
	========================================== -->
	{#if gameState === 'lobby'}
		<section in:fade={{ duration: 300 }} class="flex-1 flex flex-col items-center justify-center max-w-4xl mx-auto w-full py-8 relative z-10">
			<!-- Burning glow backdrop -->
			<div class="absolute w-[450px] h-[450px] bg-red-600/10 rounded-full blur-[100px] pointer-events-none -z-10 animate-pulse"></div>
			
			<div class="text-center mb-10">
				<!-- Hazard Stripes Border -->
				<div class="inline-block p-1 border-4 border-yellow-400 bg-black mb-4">
					<span class="text-xs text-yellow-400 tracking-widest font-black uppercase px-3 py-1">WARNING: EXTREME VOLUME REQUIRED</span>
				</div>
				<h1 class="text-5xl md:text-8xl font-black tracking-tighter uppercase text-red-500 drop-shadow-[0_0_30px_rgba(239, 68, 68, 0.6)] select-none">
					音量アップ<br class="md:hidden"><span class="text-white bg-red-600 px-3 ml-2 border-4 border-black inline-block">しりとり</span>
				</h1>
				<p class="text-xs md:text-sm text-yellow-400 mt-6 tracking-widest font-black uppercase">
					🎸 SCREAM FOR YOUR LIFE! BREAK THROUGH THE AUDIO WALL!
				</p>
			</div>

			<!-- Quick Browser Capability Check -->
			{#if !speechRecognitionSupported}
				<div class="mb-6 px-4 py-3 rounded-none border-l-4 border-yellow-400 bg-neutral-900 text-yellow-400 text-xs text-left flex items-center gap-2 max-w-md">
					<span>⚠️</span> お使いのブラウザは音声認識非対応です（Firefox, Opera 等）。マイクへのシャウト＋キーボードによるタイピング入力で死闘に参加可能です！
				</div>
			{/if}

			{#if errorMessage}
				<div class="mb-6 px-4 py-3 rounded-none border-2 border-red-500 bg-red-950/20 text-red-400 text-sm text-center max-w-lg font-black tracking-wider uppercase">
					💀 ERROR: {errorMessage}
				</div>
			{/if}

			<!-- Game Modes Grid -->
			<div class="grid grid-cols-1 md:grid-cols-2 gap-6 w-full max-w-3xl mb-10">
				
				<!-- Solo Mode Card -->
				<button 
					onclick={() => startGame('solo')}
					class="group text-left p-6 md:p-8 rounded-none bg-neutral-900 border-4 border-neutral-800 hover:border-red-500 hover:shadow-[0_0_40px_rgba(239,68,68,0.2)] transition-all duration-300 relative overflow-hidden cursor-pointer"
				>
					<div class="absolute right-0 bottom-0 translate-y-1/4 translate-x-1/4 opacity-[0.03] group-hover:opacity-[0.08] group-hover:scale-110 transition-all duration-500 text-9xl font-black font-digital text-red-500">
						SOLO
					</div>
					<div class="flex justify-between items-start mb-4">
						<span class="p-3 bg-red-600/10 text-red-500 border-2 border-red-500/30 text-2xl group-hover:scale-110 transition-transform">👤</span>
						{#if soloHighScore > 0}
							<span class="text-xs font-digital text-yellow-400 font-black bg-yellow-400/10 border border-yellow-400/20 px-2.5 py-1 uppercase tracking-wider">
								BEST SURVIVED: {soloHighScore}
							</span>
						{/if}
					</div>
					<h3 class="text-xl font-black uppercase text-white group-hover:text-red-500 transition-colors">SOLO MISSION (ソロ)</h3>
					<p class="text-sm text-neutral-400 mt-2 font-medium leading-relaxed">
						孤独な絶叫訓練！しりとりを続けながら、毎回己が叩き出した限界デシベル（dB）を更新せよ！
					</p>
				</button>

				<!-- VS Mode Card -->
				<button 
					onclick={() => startGame('vs')}
					class="group text-left p-6 md:p-8 rounded-none bg-neutral-900 border-4 border-neutral-800 hover:border-red-500 hover:shadow-[0_0_40px_rgba(239,68,68,0.2)] transition-all duration-300 relative overflow-hidden cursor-pointer"
				>
					<div class="absolute right-0 bottom-0 translate-y-1/4 translate-x-1/4 opacity-[0.03] group-hover:opacity-[0.08] group-hover:scale-110 transition-all duration-500 text-9xl font-black font-digital text-red-500">
						DUEL
					</div>
					<div class="flex justify-between items-start mb-4">
						<span class="p-3 bg-red-600/10 text-red-500 border-2 border-red-500/30 text-2xl group-hover:scale-110 transition-transform">🎸</span>
						<span class="text-xs font-black text-red-500 bg-red-500/10 border border-red-500/20 px-2.5 py-1 uppercase tracking-wider">
							SHOUT DUEL
						</span>
					</div>
					<h3 class="text-xl font-black uppercase text-white group-hover:text-red-500 transition-colors">SHOUT DUEL (対戦)</h3>
					<p class="text-sm text-neutral-400 mt-2 font-medium leading-relaxed">
						ローカル２人対戦！交互にマイクに向かって絶叫し、相手の最大声量（dB）を超えるしりとりバトル！
					</p>
				</button>

			</div>

			<!-- Microphone Setup / Calibration Card -->
			<div class="w-full max-w-xl p-6 rounded-none bg-neutral-900 border-4 border-neutral-800 flex flex-col items-center gap-4 text-center">
				<div class="flex items-center gap-2">
					<span class="w-2.5 h-2.5 rounded-none {micPermission === 'granted' ? 'bg-red-500 animate-ping shadow-[0_0_8px_#ef4444]' : 'bg-yellow-500'}"></span>
					<span class="text-xs font-black uppercase tracking-widest text-neutral-400">
						{micPermission === 'granted' ? 'SYSTEM ONLINE (MIC ACTIVE)' : 'AUDIO SYSTEM CALIBRATION'}
					</span>
				</div>

				{#if micPermission !== 'granted'}
					<button 
						onclick={initAudio}
						class="w-full md:w-auto px-6 py-3 bg-red-600 hover:bg-red-500 text-black text-xs font-black rounded-none border-2 border-black tracking-widest shadow-md hover:shadow-lg transition-all cursor-pointer uppercase"
					>
						⚡ CONNECT MICROPHONE
					</button>
				{:else}
					<div class="w-full flex flex-col items-center gap-3">
						<div class="w-full flex items-center justify-between text-xs text-neutral-400 px-1 font-bold">
							<span>🎤 AMP GAIN SENSITIVITY: x{micSensitivity.toFixed(1)}</span>
							<button 
								onclick={calibrateNoise}
								disabled={isCalibrating}
								class="text-yellow-400 hover:text-white font-black uppercase disabled:text-neutral-600 text-xs bg-transparent border-0 cursor-pointer"
							>
								{isCalibrating ? `CALIBRATING... (${calibrationCountdown})` : '🔇 CALIBRATE NOISE FLOOR'}
							</button>
						</div>
						
						<!-- Slider -->
						<div class="w-full relative flex items-center">
							<input 
								type="range" 
								min="0.5" 
								max="5.0" 
								step="0.1" 
								bind:value={micSensitivity}
								class="w-full h-2 bg-neutral-950 appearance-none cursor-pointer accent-red-500 rounded-none border border-neutral-800"
							/>
						</div>
						
						<p class="text-[10px] text-neutral-500 font-medium">
							※声量が正しく拾われない場合は、ゲインスライダーを右に引いてアンプ入力をブーストしてください。
						</p>
					</div>
				{/if}
			</div>

		</section>
	{/if}

	<!-- ==========================================
	  GAMEPLAY INTERFACE (READY, LISTENING, SHOUT LOOPS)
	========================================== -->
	{#if gameState !== 'lobby'}
		<section in:fade={{ duration: 250 }} class="flex-1 max-w-5xl mx-auto w-full flex flex-col justify-between py-6 relative z-10">
			
			<!-- A. STATS BAR & ROCKING TURN CARDS -->
			<div class="grid grid-cols-2 gap-4 mb-6">
				<!-- Player 1 Amp Box -->
				<div 
					class="p-4 rounded-none border-4 transition-all duration-300 {gameMode === 'solo' ? 'col-span-2' : ''} 
					{currentPlayer === 1 && gameState !== 'gameover' 
						? 'bg-neutral-900 border-red-500 shadow-[0_0_30px_rgba(239,68,68,0.2)] animate-pulse' 
						: 'bg-neutral-900/30 border-neutral-900 opacity-40'}"
				>
					<div class="flex justify-between items-center">
						<div class="flex items-center gap-2">
							<span class="text-lg">🎤</span>
							<span class="font-black text-sm text-white uppercase tracking-wider">
								{gameMode === 'solo' ? 'PLAYER' : 'PLAYER 1'}
							</span>
							{#if currentPlayer === 1 && gameState !== 'gameover'}
								<span class="text-[9px] bg-red-600 text-black font-black px-2 py-0.5 uppercase tracking-widest animate-bounce">ON STAGE</span>
							{/if}
						</div>
						
						<!-- Metal Horns / Skull Lives -->
						<div class="flex gap-1">
							{#each Array(3) as _, i}
								{#if i < player1Lives}
									<span class="text-xl text-red-500 transition-all duration-300 select-none scale-100 animate-bounce">🤘</span>
								{:else}
									<span class="text-xl text-neutral-850 opacity-20 transition-all duration-300 select-none scale-90">💀</span>
								{/if}
							{/each}
						</div>
					</div>
					<div class="mt-3 flex justify-between items-end">
						<span class="text-[9px] text-neutral-500 uppercase tracking-widest font-black">SOLVED CHAIN count</span>
						<span class="text-3xl font-black font-digital text-red-500 drop-shadow-[0_0_8px_rgba(239,68,68,0.4)]">{player1Score}</span>
					</div>
				</div>

				<!-- Player 2 Amp Box (VS Only) -->
				{#if gameMode === 'vs'}
					<div 
						class="p-4 rounded-none border-4 transition-all duration-300 
						{currentPlayer === 2 && gameState !== 'gameover' 
							? 'bg-neutral-900 border-red-500 shadow-[0_0_30px_rgba(239,68,68,0.2)] animate-pulse' 
							: 'bg-neutral-900/30 border-neutral-900 opacity-40'}"
					>
						<div class="flex justify-between items-center">
							<div class="flex items-center gap-2">
								<span class="text-lg">🎤</span>
								<span class="font-black text-sm text-white uppercase tracking-wider">PLAYER 2</span>
								{#if currentPlayer === 2 && gameState !== 'gameover'}
									<span class="text-[9px] bg-red-600 text-black font-black px-2 py-0.5 uppercase tracking-widest animate-bounce">ON STAGE</span>
								{/if}
							</div>
							
							<!-- Metal Horns / Skull Lives -->
							<div class="flex gap-1">
								{#each Array(3) as _, i}
									{#if i < player2Lives}
										<span class="text-xl text-red-500 transition-all duration-300 select-none scale-100 animate-bounce">🤘</span>
									{:else}
										<span class="text-xl text-neutral-850 opacity-20 transition-all duration-300 select-none scale-90">💀</span>
									{/if}
								{/each}
							</div>
						</div>
						<div class="mt-3 flex justify-between items-end">
							<span class="text-[9px] text-neutral-500 uppercase tracking-widest font-black">SOLVED CHAIN count</span>
							<span class="text-3xl font-black font-digital text-red-500 drop-shadow-[0_0_8px_rgba(239,68,68,0.4)]">{player2Score}</span>
						</div>
					</div>
				{/if}
			</div>

			<!-- B. CORE RACK VU METER -->
			<div class="flex-1 flex flex-col items-stretch justify-center my-4 gap-6">
				
				<!-- Giant Studio VU Scale -->
				<div class="w-full bg-neutral-900 border-4 border-neutral-800 p-5 shadow-2xl flex flex-col gap-4 relative">
					
					<div class="absolute top-2 left-2 w-2.5 h-2.5 bg-neutral-700 rounded-full border border-black shadow"></div>
					<div class="absolute top-2 right-2 w-2.5 h-2.5 bg-neutral-700 rounded-full border border-black shadow"></div>
					<div class="absolute bottom-2 left-2 w-2.5 h-2.5 bg-neutral-700 rounded-full border border-black shadow"></div>
					<div class="absolute bottom-2 right-2 w-2.5 h-2.5 bg-neutral-700 rounded-full border border-black shadow"></div>

					<div class="flex justify-between items-center font-digital border-b-2 border-neutral-950 pb-2">
						<span class="text-[10px] text-neutral-400 tracking-widest font-black uppercase">🎚️ MASTER VU OUT / LEVEL</span>
						<div class="flex gap-4 items-center">
							<span class="text-xs text-yellow-400 font-bold">TARGET: {targetVolume.toFixed(0)} dB</span>
							<span class="text-lg text-white font-black">{smoothedVolume.toFixed(1)} dB</span>
						</div>
					</div>

					<!-- Segmented LED Lightbar Display -->
					<div class="h-16 w-full bg-black flex gap-1 p-1 items-stretch overflow-hidden relative border-4 border-black">
						{#each Array(32) as _, i}
							{@const segmentPercent = (i / 32) * 100}
							{@const currentPercent = dbToPercent(smoothedVolume)}
							{@const isActive = currentPercent >= segmentPercent}
							
							{@const colorClass = i < 18 
								? (isActive ? 'bg-emerald-500 shadow-[0_0_12px_#10b981]' : 'bg-emerald-950/80') 
								: i < 26 
									? (isActive ? 'bg-yellow-400 shadow-[0_0_12px_#facc15]' : 'bg-yellow-950/80') 
									: (isActive ? 'bg-red-500 shadow-[0_0_15px_#ef4444] animate-pulse' : 'bg-red-950/80') 
							}
							<div class="flex-1 {colorClass} transition-all duration-75 border-r border-black/40"></div>
						{/each}

						<!-- Overlay Slide Indicator for Target Volume Limit -->
						<div 
							class="absolute top-0 bottom-0 w-2 bg-yellow-400 shadow-[0_0_20px_#facc15] border-x-2 border-black z-10 flex items-center justify-center pointer-events-none" 
							style="left: calc({dbToPercent(targetVolume)}% - 4px)"
						>
							<div class="h-5 w-5 bg-red-600 rotate-45 border-2 border-black text-[9px] flex items-center justify-center text-white font-black uppercase">
								!
							</div>
						</div>
					</div>
					
					<!-- Segment Markers Scales -->
					<div class="flex justify-between text-[9px] font-digital text-neutral-500 font-black uppercase tracking-widest px-1">
						<span>-60 dB (SILENCE)</span>
						<span>-45 dB</span>
						<span>-30 dB</span>
						<span>-15 dB</span>
						<span>-5 dB</span>
						<span>0 dB (0dBFS)</span>
						<span>+5 dB</span>
						<span class="text-red-500">+10 dB LIMIT</span>
					</div>
				</div>

				<!-- C. REALTIME VOLCANIC EMBERS CANVAS & loading dictionary bar -->
				<div class="w-full relative">
					{#if isValidatingWord}
						<div in:fade class="w-full bg-yellow-400 text-black border-4 border-black p-3 font-black text-center text-xs uppercase tracking-widest animate-pulse my-4 relative z-20">
							⚡ CRAWLING METAL DICTIONARY / 辞書確認中...
						</div>
					{/if}

					<!-- REALTIME EMBERS CANVAS -->
					<div class="w-full h-32 bg-black border-4 border-neutral-900 relative shadow-inner">
						<div class="absolute left-3 top-2 flex items-center gap-1.5 z-10 bg-black/80 px-2 py-0.5 border border-neutral-800">
							<span class="w-2 h-2 rounded-none {isListening ? 'bg-red-500 animate-ping' : 'bg-neutral-800'}"></span>
							<span class="text-[9px] uppercase tracking-widest font-black text-red-500 font-digital">
								{isListening ? '🔥 VOLCANIC FREQ WAVE ON' : '🔇 OSCILLOSCOPE STANDBY'}
							</span>
						</div>
						<canvas 
							bind:this={canvasElement} 
							width="800" 
							height="128" 
							class="w-full h-full"
						></canvas>
					</div>
				</div>
			</div>

			<!-- D. GIANT MC SHIRITORI CHAIN GRID -->
			<div class="grid grid-cols-1 md:grid-cols-3 gap-6 items-stretch bg-neutral-900 border-4 border-neutral-850 p-6 shadow-2xl relative">
				
				<!-- Previous word -->
				<div class="text-center md:text-left flex flex-col justify-center border-b-2 md:border-b-0 md:border-r-2 border-neutral-800 pb-4 md:pb-0 md:pr-4">
					<span class="text-[9px] text-neutral-500 font-black uppercase tracking-widest">PREVIOUS CHAIN / 前の言葉</span>
					<span class="text-3xl md:text-4xl font-black text-red-500 tracking-tighter uppercase mt-1 drop-shadow-[0_0_6px_rgba(239,68,68,0.2)]">{lastWord}</span>
					<span class="text-xs text-neutral-400 font-medium">（{lastWordReading}）</span>
				</div>

				<!-- Target character panel -->
				<div class="flex flex-col items-center justify-center p-4 bg-black border-2 border-neutral-800 shadow-inner">
					<span class="text-[9px] text-neutral-500 font-black uppercase tracking-widest mb-1">SHOUT WORD STARTING WITH</span>
					<div class="flex items-center gap-3">
						<span class="text-2xl font-black text-neutral-100">👉</span>
						<span class="text-5xl font-black text-black bg-yellow-400 border-4 border-black px-4 py-1.5 leading-none font-sans drop-shadow-[0_0_12px_rgba(250,204,21,0.4)]">
							{getEndChar(lastWordReading)}
						</span>
					</div>
				</div>

				<!-- Action Buttons -->
				<div class="flex flex-col gap-2 justify-center">
					{#if gameState === 'ready'}
						<button 
							onclick={startListening}
							class="w-full py-4 bg-red-600 hover:bg-red-500 text-black font-black uppercase tracking-widest text-sm border-2 border-black hover:scale-[1.02] shadow-[0_0_25px_rgba(239,68,68,0.4)] active:scale-95 transition-all flex items-center justify-center gap-2 cursor-pointer"
						>
							🎤 HOLD SPACE & SHOUT! (スペース長押しでシャウト)
						</button>
						<p class="text-[9px] text-center text-neutral-500 font-bold uppercase tracking-wider">
							PUSH AND HOLD <span class="bg-neutral-800 px-1.5 py-0.5 rounded-none text-neutral-300 font-mono text-[9px]">SPACEBAR</span> TO RECORD, RELEASE TO SOLVE
						</p>
					{:else if gameState === 'listening'}
						<button 
							onclick={stopListening}
							class="w-full py-4 bg-black text-red-500 font-black uppercase tracking-widest text-sm border-2 border-red-500 hover:bg-red-950/20 active:scale-95 transition-all flex items-center justify-center gap-2 cursor-pointer"
						>
							⏹️ RELEASE SPACE TO SOLVE (スペースを離して送信)
						</button>
						<p class="text-[9px] text-center text-red-500 font-black uppercase tracking-widest animate-pulse">
							{#if recognizedText}
								🎙️ CAPTURED: 「<span class="text-white bg-red-600 px-1.5">{recognizedText}</span>」
							{:else}
								🎙️ SCREAM NOW! (スペースを離すと解読開始します)
							{/if}
						</p>
					{/if}

					<!-- Skip / Reset Button -->
					{#if gameState === 'ready'}
						<button 
							onclick={cowardsReset}
							class="w-full py-2 border border-neutral-800 hover:border-red-500 hover:text-red-500 text-neutral-500 text-[10px] font-black uppercase tracking-widest transition-colors flex items-center justify-center gap-1 cursor-pointer bg-transparent"
						>
							☠️ COWARD'S PASS (SACRIFICE 1 HORN LIFE TO RESET TARGET)
						</button>
					{/if}
				</div>
			</div>

		</section>
	{/if}

	<!-- ==========================================
	  TEMPORARY TRANSITION STATE BANNERS
	========================================== -->

	<!-- A. TEMPORARY SUCCESS STATE BANNER -->
	{#if gameState === 'success'}
		<div in:fade={{ duration: 150 }} class="fixed inset-0 bg-neutral-950/95 backdrop-blur-sm flex flex-col items-center justify-center z-50 text-center p-4 border-8 border-red-600">
			<div in:scale={{ duration: 300 }} class="flex flex-col items-center gap-4">
				<span class="text-7xl animate-bounce">🤘🔥🤘</span>
				<h1 class="text-6xl md:text-8xl font-black text-white bg-red-600 px-6 py-2 border-4 border-black uppercase tracking-wider drop-shadow-[0_0_20px_rgba(239,68,68,0.8)] font-digital">
					STAGE CLEAR!
				</h1>
				<p class="text-lg text-neutral-200 mt-4 max-w-lg font-black uppercase tracking-widest">
					しりとり成立！音圧爆破クリア！<br>
					NEXT SHOUT TARGET IS INCRIMINATED TO:<br class="md:hidden">
					<span class="text-yellow-400 text-3xl font-digital ml-1 font-black underline">{targetVolume.toFixed(1)} dB</span>
				</p>
			</div>
		</div>
	{/if}

	<!-- B. TEMPORARY FAILURE STATE BANNER -->
	{#if gameState === 'failure'}
		<div in:fade={{ duration: 150 }} class="fixed inset-0 bg-neutral-950/95 backdrop-blur-sm flex flex-col items-center justify-center z-50 text-center p-4 border-8 border-yellow-500">
			<div in:scale={{ duration: 300 }} class="flex flex-col items-center gap-4">
				<span class="text-7xl">💀☠️💀</span>
				<h1 class="text-5xl md:text-7xl font-black text-black bg-yellow-400 px-6 py-2 border-4 border-black uppercase tracking-wider font-digital">
					REJECTED!
				</h1>
				<p class="text-lg text-neutral-300 mt-4 max-w-lg font-black uppercase tracking-wider text-yellow-400">
					{errorMessage}
				</p>
				<p class="text-xs text-neutral-500 font-black mt-2 uppercase tracking-widest">
					HORN LIFE DEDUCTED. WEAK VOICE HAS BEEN CONDEMNED.
				</p>
			</div>
		</div>
	{/if}

	<!-- C. RETRO ARCADE GAME OVER SCREEN -->
	{#if gameState === 'gameover'}
		<div in:fade={{ duration: 350 }} class="fixed inset-0 bg-black flex items-center justify-center p-4 z-50 text-center border-8 border-double border-red-600">
			<div in:scale={{ duration: 300 }} class="max-w-lg w-full flex flex-col gap-6 p-8 rounded-none bg-neutral-900 border-4 border-black relative">
				
				<div class="absolute top-2 left-2 w-2.5 h-2.5 bg-neutral-700 rounded-full border border-black shadow"></div>
				<div class="absolute top-2 right-2 w-2.5 h-2.5 bg-neutral-700 rounded-full border border-black shadow"></div>
				
				<div class="flex flex-col gap-2">
					<span class="text-6xl animate-pulse">💀</span>
					<h1 class="text-5xl md:text-7xl font-black text-red-500 uppercase tracking-widest font-digital drop-shadow-[0_0_15px_#ef4444]">
						DEFEATED
					</h1>
					<p class="text-xs text-neutral-400 tracking-widest font-black uppercase">THE SHIRITORI METAL CHAIN HAS BEEN BROKEN!</p>
				</div>

				<!-- Game Stats Summary -->
				<div class="p-6 bg-black border-2 border-neutral-800 flex flex-col gap-4 text-left">
					
					<div class="flex justify-between items-center pb-3 border-b border-neutral-800">
						<span class="text-xs text-neutral-400 font-black uppercase tracking-wider">TOTAL ROUNDS PLAYED</span>
						<span class="text-lg font-black font-digital text-white">{roundsCount} SURVIVED</span>
					</div>

					{#if gameMode === 'solo'}
						<div class="flex justify-between items-center">
							<span class="text-xs text-neutral-400 font-black uppercase tracking-wider">CHAIN COUNT SCORE</span>
							<span class="text-2xl font-black font-digital text-red-500">{player1Score}</span>
						</div>
					{:else}
						<div class="flex flex-col gap-2">
							<div class="flex justify-between items-center text-xs text-neutral-400 font-bold">
								<span>P1 METAL SCORE:</span>
								<span class="font-digital text-sm font-black text-white">{player1Score}</span>
							</div>
							<div class="flex justify-between items-center text-xs text-neutral-400 font-bold">
								<span>P2 METAL SCORE:</span>
								<span class="font-digital text-sm font-black text-white">{player2Score}</span>
							</div>
							<div class="flex justify-between items-center pt-2 border-t border-neutral-850 font-black text-sm">
								<span class="text-neutral-400 uppercase">GLORIOUS WINNER:</span>
								{#if player1Score === player2Score}
									<span class="text-yellow-400 uppercase">MUTUAL DRAW 🤝</span>
								{:else}
									<span class="text-red-500 uppercase">
										{player1Score > player2Score ? 'PLAYER 1 🏆' : 'PLAYER 2 🏆'}
									</span>
								{/if}
							</div>
						</div>
					{/if}

					<div class="flex justify-between items-center pt-3 border-t border-neutral-800">
						<span class="text-xs text-neutral-400 font-black uppercase tracking-wider">TERMINAL TARGET LEVEL</span>
						<span class="text-xl font-black font-digital text-yellow-400">{targetVolume.toFixed(0)} dB</span>
					</div>
				</div>

				<!-- Actions -->
				<div class="flex flex-col gap-2">
					<button 
						onclick={() => startGame(gameMode)}
						class="py-4 bg-red-600 hover:bg-red-500 text-black font-black uppercase tracking-widest text-sm border-2 border-black hover:scale-[1.02] shadow-[0_0_20px_rgba(239,68,68,0.3)] active:scale-95 transition-all cursor-pointer"
					>
						🎮 START RE-MATCH
					</button>
					<button 
						onclick={returnToLobby}
						class="py-3 border-2 border-neutral-800 hover:border-neutral-700 text-xs text-neutral-400 hover:text-white uppercase font-black tracking-widest cursor-pointer bg-transparent"
					>
						🔙 RETREAT TO LOBBY
					</button>
				</div>
			</div>
		</div>
	{/if}

	<!-- D. RULE & HELP MODAL -->
	{#if showHelpModal}
		<div class="fixed inset-0 bg-black/95 backdrop-blur-sm flex items-center justify-center p-4 z-50 overflow-y-auto">
			<div 
				in:scale={{ start: 0.95, duration: 200 }} 
				class="w-full max-w-xl bg-neutral-900 border-4 border-neutral-800 p-6 md:p-8 flex flex-col gap-5 shadow-[0_0_50px_rgba(0,0,0,0.9)]"
			>
				<div class="text-center border-b-2 border-neutral-800 pb-3">
					<h3 class="text-2xl font-black text-red-500 uppercase tracking-widest">📖 BATTLE RULES / 凶暴ルール</h3>
				</div>

				<div class="text-sm text-neutral-300 flex flex-col gap-4 font-bold tracking-wider leading-relaxed">
					<div class="flex gap-3">
						<span class="text-lg">🎸</span>
						<p><strong class="text-yellow-400 uppercase">1. HOLD SPACE & SCREAM!</strong><br>
						スペースキーを押している間だけマイク録音が走ります。話し終わったらスペースキーを離すことで瞬時に解読されます！同じ言葉は二度と使えません！</p>
					</div>

					<div class="flex gap-3">
						<span class="text-lg">🔗</span>
						<p><strong class="text-yellow-400 uppercase">2. BATTLE CHAIN CHECK</strong><br>
						音声認識により繋ぎ判定を行います。「ん」で終わる単語は即時ゲームオーバーとなります。※認識不能な難読漢字ワードの場合は、判定が自動バイパスされて次へ引き継がれます！</p>
					</div>

					<div class="flex gap-3">
						<span class="text-lg">🎚️</span>
						<p><strong class="text-yellow-400 uppercase">3. BEAT THE DECIBELS (最重要)</strong><br>
						しりとりが成立していても、<strong>前回のプレイヤーが叩き出した最大声量（ターゲットdB）を超える大声</strong>で絶叫しなければ即却下され、ライフを消費します！</p>
					</div>

					<div class="flex gap-3">
						<span class="text-lg">📈</span>
						<p><strong class="text-yellow-400 uppercase">4. TARGET POWER INCREMENT</strong><br>
						クリアに成功すると、あなたが発声した最大声量（dB）が「次のプレイヤーの目標値」に上書きされます！ゲームが進むほど必要声量は高くなります！</p>
					</div>

					<div class="flex gap-3">
						<span class="text-lg">☠️</span>
						<p><strong class="text-yellow-400 uppercase">5. COWARD'S PASS</strong><br>
						目標デシベルが高すぎて声が出ない場合は、ライフを1枚犠牲にして<strong>目標音量をベースライン（-50 dB）にリセット</strong>する「パス」が可能です！</p>
					</div>
				</div>

				<button 
					onclick={() => showHelpModal = false}
					class="w-full py-3 bg-neutral-850 hover:bg-neutral-800 border-2 border-neutral-700 hover:border-neutral-600 text-white font-black text-xs uppercase tracking-widest transition-colors mt-3 cursor-pointer"
				>
					🆗 ROCK AND ROLL! (了解)
				</button>
			</div>
		</div>
	{/if}

	<!-- ==========================================
	  FOOTER SECTION
	========================================== -->
	<footer class="w-full text-center relative z-10 py-2 border-t-2 border-neutral-900">
		<span class="text-[9px] tracking-widest text-neutral-600 font-black uppercase font-digital">
			HELL OUT AUDIO ENGINE v2.0 • POWERED BY WEBAUDIO & WEBSPEECH APIS
		</span>
	</footer>

</main>

<style>
	/* Background container style overrides */
	:global(body) {
		margin: 0;
		padding: 0;
		background-color: #0A0A0A;
		overflow-x: hidden;
	}
</style>
