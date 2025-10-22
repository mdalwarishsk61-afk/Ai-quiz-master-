<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI General Knowledge Quiz</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800&display=swap" rel="stylesheet">
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc, onSnapshot } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global Firebase variables will be defined here (outside the IIFE)
        window.app = null;
        window.db = null;
        window.auth = null;
        window.userId = null;
        window.gameState = {
            currentLevel: 1,
            currentScore: 0,
            questionIndex: 0, 
            totalQuestionsForLevel: 3,
            seenQuestionIds: [],
            currentQuestion: null,
            isProcessing: false,
            isGameRunning: false,
            lives: 3,
            timerInterval: null,
            currentLanguage: 'en'
        };

        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-gk-quiz-app-id';

        // --- Translation Data ---
        const translations = {
            en: {
                level: "LEVEL", score: "SCORE",
                mainTitle: "AI Knowledge Test",
                startScreenText: "You are on <strong>Level {level}</strong>.<br> Complete {qCount} questions to advance.",
                startScreenInfo: "You have 3 lives. One mistake costs a life. The app uses AI to generate new questions, ensuring endless play.",
                chooseLang: "Choose Your Language",
                loadingAI: "Generating Unique Question...",
                questionCounter: "Question {index} of {total}",
                gameOverTitle: "GAME OVER!",
                gameOverText: "You've run out of lives. Better luck next time!",
                gameOverScore: "Final Score",
                restartingLevel: "Restarting Level",
                retryButton: "TRY AGAIN",
                levelCompleteTitle: "LEVEL CLEARED!",
                levelCompleteText: "Fantastic knowledge! Moving up.",
                totalScore: "Total Score",
                nextLevel: "Next Level",
                nextLevelButton: "START NEXT LEVEL"
            },
            hi: {
                level: "स्तर", score: "स्कोर",
                mainTitle: "एआई ज्ञान परीक्षण",
                startScreenText: "आप <strong>स्तर {level}</strong> पर हैं।<br> आगे बढ़ने के लिए {qCount} प्रश्न पूरे करें।",
                startScreenInfo: "आपके पास 3 जीवन हैं। एक गलती पर एक जीवन जाएगा। यह ऐप नए प्रश्न बनाने के लिए एआई का उपयोग करता है, जिससे अंतहीन खेल सुनिश्चित होता है।",
                chooseLang: "अपनी भाषा चुनें",
                loadingAI: "अद्वितीय प्रश्न बनाया जा रहा है...",
                questionCounter: "प्रश्न {index} / {total}",
                gameOverTitle: "खेल खत्म!",
                gameOverText: "आपके जीवन समाप्त हो गए हैं। अगली बार बेहतर किस्मत!",
                gameOverScore: "अंतिम स्कोर",
                restartingLevel: "स्तर फिर से शुरू हो रहा है",
                retryButton: "पुनः प्रयास करें",
                levelCompleteTitle: "स्तर पूरा हुआ!",
                levelCompleteText: "शानदार ज्ञान! आगे बढ़ते हैं।",
                totalScore: "कुल स्कोर",
                nextLevel: "अगला स्तर",
                nextLevelButton: "अगला स्तर शुरू करें"
            },
            bn: {
                level: "স্তর", score: "স্কোর",
                mainTitle: "এআই জ্ঞান পরীক্ষা",
                startScreenText: "আপনি <strong>স্তর {level}</strong>-এ আছেন।<br> এগিয়ে যেতে {qCount}টি প্রশ্ন সম্পন্ন করুন।",
                startScreenInfo: "আপনার ৩টি জীবন আছে। একটি ভুলে একটি জীবন কাটা যাবে। অ্যাপটি নতুন প্রশ্ন তৈরি করতে এআই ব্যবহার করে, যা অফুরন্ত খেলা নিশ্চিত করে।",
                chooseLang: "আপনার ভাষা নির্বাচন করুন",
                loadingAI: "অনন্য প্রশ্ন তৈরি করা হচ্ছে...",
                questionCounter: "প্রশ্ন {index} / {total}",
                gameOverTitle: "খেলা শেষ!",
                gameOverText: "আপনার সব জীবন শেষ। পরেরবার ভাগ্য আপনার সহায় হোক!",
                gameOverScore: "চূড়ান্ত স্কোর",
                restartingLevel: "স্তর পুনরায় শুরু হচ্ছে",
                retryButton: "আবার চেষ্টা করুন",
                levelCompleteTitle: "স্তর সম্পন্ন!",
                levelCompleteText: "অসাধারণ জ্ঞান! এগিয়ে চলুন।",
                totalScore: "মোট স্কোর",
                nextLevel: "পরবর্তী স্তর",
                nextLevelButton: "পরবর্তী স্তর শুরু করুন"
            },
            ta: {
                level: "நிலை", score: "மதிப்பெண்",
                mainTitle: "AI அறிவு சோதனை",
                startScreenText: "நீங்கள் <strong>நிலை {level}</strong>-ல் இருக்கிறீர்கள்.<br> முன்னேற {qCount} கேள்விகளை முடிக்கவும்.",
                startScreenInfo: "உங்களுக்கு 3 உயிர்கள் உள்ளன. ஒரு தவறுக்கு ஒரு உயிர் போகும். இந்த ஆப் புதிய கேள்விகளை உருவாக்க AI-ஐப் பயன்படுத்துகிறது, முடிவற்ற விளையாட்டை உறுதி செய்கிறது.",
                chooseLang: "உங்கள் மொழியைத் தேர்ந்தெடுக்கவும்",
                loadingAI: "தனித்துவமான கேள்வி உருவாக்கப்படுகிறது...",
                questionCounter: "கேள்வி {index} / {total}",
                gameOverTitle: "விளையாட்டு முடிந்தது!",
                gameOverText: "உங்கள் உயிர்கள் தீர்ந்துவிட்டன. அடுத்த முறை късмет!",
                gameOverScore: "இறுதி மதிப்பெண்",
                restartingLevel: "நிலை மீண்டும் தொடங்குகிறது",
                retryButton: "மீண்டும் চেষ্টা செய்",
                levelCompleteTitle: "நிலை முடிந்தது!",
                levelCompleteText: "அద్భుதமான அறிவு! மேலே செல்லுங்கள்.",
                totalScore: "மொத்த மதிப்பெண்",
                nextLevel: "அடுத்த நிலை",
                nextLevelButton: "அடுத்த நிலையைத் தொடங்கு"
            }
        };

        const getQuestionsPerLevel = (level) => (2 * level) + 1;

        // Firebase Initialization and Authentication
        const initFirebase = async () => {
            try {
                if (window.app) return; 

                setLogLevel('error'); 
                window.app = initializeApp(firebaseConfig);
                window.db = getFirestore(window.app);
                window.auth = getAuth(window.app);

                if (initialAuthToken) {
                    await signInWithCustomToken(window.auth, initialAuthToken);
                } else {
                    await signInAnonymously(window.auth);
                }

                onAuthStateChanged(window.auth, (user) => {
                    if (user) {
                        window.userId = user.uid;
                        loadProgressAndSetupGame();
                    } else {
                        window.userId = crypto.randomUUID();
                        loadProgressAndSetupGame();
                    }
                });
            } catch (error) {
                console.error("Firebase Initialization Error:", error);
            }
        };

        const getProgressDocRef = () => {
            if (!window.db || !window.userId) return null;
            const path = `/artifacts/${appId}/users/${window.userId}/quiz_progress/data`;
            return doc(window.db, path);
        };

        const loadProgressAndSetupGame = () => {
            const ref = getProgressDocRef();
            if (!ref) return;

            onSnapshot(ref, (docSnap) => {
                const splashScreen = document.getElementById('splash-screen');
                splashScreen.classList.add('hidden');
                document.getElementById('game-container').classList.remove('hidden');

                if (docSnap.exists()) {
                    const data = docSnap.data();
                    window.gameState.currentLevel = data.currentLevel || 1;
                    window.gameState.currentScore = data.currentScore || 0;
                    window.gameState.seenQuestionIds = data.seenQuestionIds || [];
                }

                if (!window.gameState.isGameRunning) {
                    showScreen('start-screen');
                    setLanguage(window.gameState.currentLanguage); 
                }

            }, (error) => {
                console.error("Error loading progress:", error);
                showScreen('start-screen'); 
            });
        };

        window.saveProgress = async () => {
            const ref = getProgressDocRef();
            if (!ref) return;
            try {
                await setDoc(ref, {
                    userId: window.userId,
                    currentLevel: window.gameState.currentLevel,
                    currentScore: window.gameState.currentScore,
                    seenQuestionIds: window.gameState.seenQuestionIds,
                    lastPlayed: new Date()
                }, { merge: true });
            } catch (error) {
                console.error("Error saving progress:", error);
            }
        };

        const AudioContext = window.AudioContext || window.webkitAudioContext;
        window.audioContext = AudioContext ? new AudioContext() : null;
        
        window.playSound = (frequency, duration, type = 'sine') => {
            if (!window.audioContext) return;
            const oscillator = window.audioContext.createOscillator();
            const gainNode = window.audioContext.createGain();
            oscillator.connect(gainNode);
            gainNode.connect(window.audioContext.destination);
            oscillator.frequency.setValueAtTime(frequency, window.audioContext.currentTime);
            oscillator.type = type;
            gainNode.gain.setValueAtTime(0, window.audioContext.currentTime);
            gainNode.gain.linearRampToValueAtTime(0.5, window.audioContext.currentTime + 0.01);
            gainNode.gain.linearRampToValueAtTime(0.0001, window.audioContext.currentTime + 0.01 + duration);
            oscillator.start();
            setTimeout(() => { oscillator.stop(); }, duration * 1000);
        };

        window.setLanguage = (lang) => {
            window.gameState.currentLanguage = lang;
            const t = translations[lang];
            document.querySelectorAll('[data-key]').forEach(el => {
                const key = el.dataset.key;
                if (t[key]) {
                    el.innerHTML = t[key];
                }
            });
            updateHeaderUI();
            updateStartScreenText();
        }

        window.updateStartScreenText = () => {
            const t = translations[window.gameState.currentLanguage];
            const level = window.gameState.currentLevel;
            const qCount = getQuestionsPerLevel(level);
            document.getElementById('start-level-text').innerHTML = t.startScreenText
                .replace('{level}', level)
                .replace('{qCount}', qCount);
        };

        window.updateHeaderUI = () => {
            const t = translations[window.gameState.currentLanguage];
            document.getElementById('level-label').textContent = `${t.level}:`;
            document.getElementById('level-info').textContent = window.gameState.currentLevel;
            document.getElementById('score-label').textContent = `${t.score}:`;
            document.getElementById('score-info').textContent = window.gameState.currentScore;
            updateLivesDisplay();
        };

        window.updateLivesDisplay = () => {
            const livesContainer = document.getElementById('lives-container');
            livesContainer.innerHTML = '';
            for (let i = 0; i < window.gameState.lives; i++) {
                livesContainer.innerHTML += '❤️';
            }
             for (let i = window.gameState.lives; i < 3; i++) {
                livesContainer.innerHTML += '🖤';
            }
        };

        window.showScreen = (screenId) => {
            const screens = ['splash-screen', 'start-screen', 'question-screen', 'game-over-screen', 'level-complete-screen', 'loading-ai-screen'];
            screens.forEach(id => document.getElementById(id)?.classList.add('hidden'));
            document.getElementById(screenId)?.classList.remove('hidden');
        };

        window.selectLanguageAndStart = (lang) => {
            setLanguage(lang);
            startGame();
        }

        window.startGame = () => {
            window.gameState.isGameRunning = true;
            window.gameState.questionIndex = 0;
            window.gameState.lives = 3;
            window.gameState.totalQuestionsForLevel = getQuestionsPerLevel(window.gameState.currentLevel);
            updateHeaderUI();
            startNextQuestion();
        };

        const startNextQuestion = async () => {
            window.gameState.questionIndex++;
            showScreen('loading-ai-screen');
            window.gameState.isProcessing = true;
            
            document.querySelectorAll('.option-button').forEach(btn => {
                btn.className = 'option-button w-full p-4 text-left bg-white/10 text-white/90 rounded-xl hover:bg-white/20 transition duration-150 shadow-lg';
                btn.disabled = true;
            });
            
            try {
                window.gameState.currentQuestion = await getUniqueQuestion();
                showQuestion(window.gameState.currentQuestion);
                window.gameState.isProcessing = false;
            } catch (error) {
                console.error("Error starting question:", error);
                gameOver();
            }
        };

        window.startTimer = () => {
            clearInterval(window.gameState.timerInterval);
            let timeLeft = 30;
            const timerEl = document.getElementById('timer-countdown');
            timerEl.textContent = timeLeft;
            timerEl.style.color = 'white';

            window.gameState.timerInterval = setInterval(() => {
                timeLeft--;
                timerEl.textContent = timeLeft;
                if (timeLeft <= 10) timerEl.style.color = '#ef4444'; // Red color
                if (timeLeft <= 0) {
                    handleTimeUp();
                }
            }, 1000);
        };
        
        window.handleTimeUp = () => {
            clearInterval(window.gameState.timerInterval);
            window.playSound(200, 0.5, 'sawtooth');
            handleIncorrectAnswer();
        };

        window.handleAnswerClick = (button, selectedAnswer) => {
            if (window.gameState.isProcessing) return;
            window.gameState.isProcessing = true;
            clearInterval(window.gameState.timerInterval);

            const correctAnswerText = window.gameState.currentQuestion.correctAnswer;
            const allButtons = document.querySelectorAll('.option-button');
            allButtons.forEach(btn => btn.disabled = true);
            const correctAnswerButton = Array.from(allButtons).find(btn => btn.dataset.answer === correctAnswerText);

            if (selectedAnswer === correctAnswerText) {
                window.playSound(600, 0.1, 'square');
                button.classList.add('bg-green-600', 'text-white', 'scale-105');
                window.gameState.currentScore++;
                document.getElementById('score-info').textContent = window.gameState.currentScore;
                
                setTimeout(() => {
                    if (window.gameState.questionIndex >= window.gameState.totalQuestionsForLevel) {
                        levelComplete();
                    } else {
                        startNextQuestion();
                    }
                }, 1000);

            } else {
                window.playSound(200, 0.5, 'sawtooth');
                button.classList.add('bg-red-600', 'text-white');
                if (correctAnswerButton) {
                    correctAnswerButton.classList.add('bg-green-500', 'text-white');
                }
                handleIncorrectAnswer(true);
            }
            window.saveProgress();
        };
        
        const handleIncorrectAnswer = (isWrongAnswerClick = false) => {
            window.gameState.lives--;
            updateLivesDisplay();

            setTimeout(() => {
                if (window.gameState.lives <= 0) {
                    gameOver();
                } else {
                    if (window.gameState.questionIndex >= window.gameState.totalQuestionsForLevel) {
                         levelComplete();
                    } else {
                         startNextQuestion();
                    }
                }
            }, isWrongAnswerClick ? 2000 : 1000); 
        };

        const showQuestion = (questionData) => {
            showScreen('question-screen');
            const t = translations[window.gameState.currentLanguage];
            document.getElementById('question-text').textContent = questionData.question;
            document.getElementById('question-counter').textContent = t.questionCounter
                .replace('{index}', window.gameState.questionIndex)
                .replace('{total}', window.gameState.totalQuestionsForLevel);

            const optionElements = document.querySelectorAll('.option-button');
            const labels = ['A', 'B', 'C', 'D'];
            questionData.options.forEach((optionText, index) => {
                const btn = optionElements[index];
                btn.textContent = `${labels[index]}. ${optionText}`;
                btn.dataset.answer = optionText;
                btn.onclick = () => window.handleAnswerClick(btn, optionText);
                btn.disabled = false;
            });
            window.gameState.isProcessing = false;
            startTimer();
        };

        window.gameOver = () => {
            window.gameState.isGameRunning = false;
            document.getElementById('game-over-level').textContent = window.gameState.currentLevel;
            document.getElementById('game-over-score').textContent = window.gameState.currentScore;
            showScreen('game-over-screen');
            setLanguage(window.gameState.currentLanguage);
        };

        window.levelComplete = () => {
            window.playSound(800, 0.2, 'square');
            window.gameState.isGameRunning = false;
            window.gameState.currentLevel++;
            window.gameState.questionIndex = 0;
            window.saveProgress();
            document.getElementById('next-level-number').textContent = window.gameState.currentLevel;
            document.getElementById('next-level-score').textContent = window.gameState.currentScore;
            showScreen('level-complete-screen');
            setLanguage(window.gameState.currentLanguage);
        };
        
        window.returnToHome = () => {
            window.gameState.isGameRunning = false;
            showScreen('start-screen');
            setLanguage(window.gameState.currentLanguage);
        };

        const STATIC_QUESTIONS = {
             en: [
                { id: 'S1', level: 1, question: "What is the largest planet in our solar system?", options: ["Mars", "Jupiter", "Saturn", "Earth"], correctAnswer: "Jupiter" },
                { id: 'S2', level: 1, question: "Which chemical element has the symbol 'O'?", options: ["Gold", "Oxygen", "Osmium", "Iron"], correctAnswer: "Oxygen" },
                { id: 'S3', level: 1, question: "Who painted the Mona Lisa?", options: ["Vincent van Gogh", "Pablo Picasso", "Leonardo da
