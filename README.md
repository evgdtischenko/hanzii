--- src/App.tsx (原始)
import { useState, useEffect, useCallback, useRef } from 'react';

interface Word {
  id: string;
  chinese: string;
  pinyin: string;
  translation: string;
}

type Mode = 'add' | 'cards' | 'quiz' | 'write';

function App() {
  const [words, setWords] = useState<Word[]>(() => {
    const saved = localStorage.getItem('chinese-words');
    return saved ? JSON.parse(saved) : [];
  });
  const [mode, setMode] = useState<Mode>('add');

  // Add word form
  const [chinese, setChinese] = useState('');
  const [pinyin, setPinyin] = useState('');
  const [translation, setTranslation] = useState('');

  // Bulk add
  const [bulkText, setBulkText] = useState('');
  const [bulkPreview, setBulkPreview] = useState<Word[]>([]);

  // Cards mode
  const [cardQueue, setCardQueue] = useState<number[]>([]);
  const [cardIndex, setCardIndex] = useState(0);
  const [isFlipped, setIsFlipped] = useState(false);
  const [showPinyin, setShowPinyin] = useState(true);
  const [knownCount, setKnownCount] = useState(0);
  const [unknownCount, setUnknownCount] = useState(0);
  const [swipeDirection, setSwipeDirection] = useState<'left' | 'right' | null>(null);
  const [touchStart, setTouchStart] = useState<number | null>(null);
  const [touchDelta, setTouchDelta] = useState(0);

  // Quiz mode
  const [quizAnswer, setQuizAnswer] = useState('');
  const [quizResult, setQuizResult] = useState<'correct' | 'wrong' | null>(null);
  const [quizScore, setQuizScore] = useState(0);
  const [quizTotal, setQuizTotal] = useState(0);
  const [quizOrder, setQuizOrder] = useState<number[]>([]);
  const [quizIndex, setQuizIndex] = useState(0);

  // Write mode
  const [writeInput, setWriteInput] = useState('');
  const [writeResult, setWriteResult] = useState<'correct' | 'wrong' | null>(null);
  const [showHint, setShowHint] = useState(false);
  const [writeOrder, setWriteOrder] = useState<number[]>([]);
  const [writeIndex, setWriteIndex] = useState(0);
  const [writeScore, setWriteScore] = useState(0);
  const [writeTotal, setWriteTotal] = useState(0);

  const modeRef = useRef(mode);
  const wordsLenRef = useRef(words.length);
  const isFlippedRef = useRef(isFlipped);

  useEffect(() => { modeRef.current = mode; }, [mode]);
  useEffect(() => { wordsLenRef.current = words.length; }, [words.length]);
  useEffect(() => { isFlippedRef.current = isFlipped; }, [isFlipped]);

  useEffect(() => {
    localStorage.setItem('chinese-words', JSON.stringify(words));
  }, [words]);

  useEffect(() => {
    if (mode === 'quiz' && words.length > 0) {
      const order = Array.from({ length: words.length }, (_, i) => i);
      for (let i = order.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [order[i], order[j]] = [order[j], order[i]];
      }
      setQuizOrder(order);
      setQuizIndex(0);
      setQuizAnswer('');
      setQuizResult(null);
    }
    if (mode === 'write' && words.length > 0) {
      const order = Array.from({ length: words.length }, (_, i) => i);
      for (let i = order.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [order[i], order[j]] = [order[j], order[i]];
      }
      setWriteOrder(order);
      setWriteIndex(0);
      setWriteInput('');
      setWriteResult(null);
      setShowHint(false);
    }
    if (mode === 'cards' && words.length > 0 && cardQueue.length === 0) {
      startCardsSession();
    }
  }, [mode, words.length]);

  const addWord = () => {
    if (!chinese.trim() || !translation.trim()) return;
    const newWord: Word = {
      id: Date.now().toString(),
      chinese: chinese.trim(),
      pinyin: pinyin.trim(),
      translation: translation.trim(),
    };
    setWords([...words, newWord]);
    setChinese('');
    setPinyin('');
    setTranslation('');
  };

  const deleteWord = (id: string) => {
    setWords(words.filter(w => w.id !== id));
  };

  // Parse bulk text into words
  const parseBulkText = (text: string): Word[] => {
    const lines = text.split('\n').filter(line => line.trim());
    const parsed: Word[] = [];

    lines.forEach((line, idx) => {
      // Try different separators: tab, " - ", " = ", " → ", " → ", ","
      let parts: string[] = [];

      if (line.includes('\t')) {
        parts = line.split('\t').map(s => s.trim()).filter(Boolean);
      } else if (line.includes(' - ')) {
        parts = line.split(' - ').map(s => s.trim()).filter(Boolean);
      } else if (line.includes(' = ')) {
        parts = line.split(' = ').map(s => s.trim()).filter(Boolean);
      } else if (line.includes(' → ')) {
        parts = line.split(' → ').map(s => s.trim()).filter(Boolean);
      } else if (line.includes('->')) {
        parts = line.split('->').map(s => s.trim()).filter(Boolean);
      } else if (line.includes(',')) {
        parts = line.split(',').map(s => s.trim()).filter(Boolean);
      } else {
        // Try splitting by spaces — first word is Chinese, rest is translation
        const trimmed = line.trim();
        const spaceIdx = trimmed.indexOf(' ');
        if (spaceIdx > 0) {
          parts = [trimmed.slice(0, spaceIdx), trimmed.slice(spaceIdx + 1).trim()];
        } else {
          parts = [trimmed];
        }
      }

      if (parts.length >= 2) {
        parsed.push({
          id: `bulk-${Date.now()}-${idx}`,
          chinese: parts[0],
          pinyin: parts.length >= 3 ? parts[1] : '',
          translation: parts.length >= 3 ? parts[2] : parts[1],
        });
      }
    });

    return parsed;
  };

  const handleBulkChange = (text: string) => {
    setBulkText(text);
    setBulkPreview(parseBulkText(text));
  };

  const addBulkWords = () => {
    if (bulkPreview.length === 0) return;
    setWords([...words, ...bulkPreview]);
    setBulkText('');
    setBulkPreview([]);
  };

  const shuffleArray = (arr: number[]): number[] => {
    const result = [...arr];
    for (let i = result.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [result[i], result[j]] = [result[j], result[i]];
    }
    return result;
  };

  const startCardsSession = useCallback(() => {
    const indices = Array.from({ length: words.length }, (_, i) => i);
    setCardQueue(shuffleArray(indices));
    setCardIndex(0);
    setIsFlipped(false);
    setKnownCount(0);
    setUnknownCount(0);
    setSwipeDirection(null);
  }, [words.length]);

  const swipeRight = useCallback(() => {
    // Помню - убираем карточку
    setSwipeDirection('right');
    setTimeout(() => {
      setKnownCount(prev => prev + 1);
      setCardIndex(prev => prev + 1);
      setIsFlipped(false);
      setSwipeDirection(null);
    }, 300);
  }, []);

  const swipeLeft = useCallback(() => {
    // Не помню - возвращаем карточку в конец очереди
    setSwipeDirection('left');
    setTimeout(() => {
      setUnknownCount(prev => prev + 1);
      const currentCardWordIndex = cardQueue[cardIndex];
      // Удаляем текущую карточку и добавляем в конец
      const newQueue = [...cardQueue.slice(0, cardIndex), ...cardQueue.slice(cardIndex + 1), currentCardWordIndex];
      setCardQueue(newQueue);
      // Индекс остаётся тем же, т.к. мы удалили элемент перед ним
      setIsFlipped(false);
      setSwipeDirection(null);
    }, 300);
  }, [cardQueue, cardIndex]);

  const checkQuizAnswer = () => {
    if (!quizAnswer.trim() || quizOrder.length === 0) return;
    const currentWord = words[quizOrder[quizIndex]];
    if (!currentWord) return;
    const isCorrect = quizAnswer.trim().toLowerCase() === currentWord.translation.toLowerCase();
    setQuizResult(isCorrect ? 'correct' : 'wrong');
    setQuizTotal(prev => prev + 1);
    if (isCorrect) setQuizScore(prev => prev + 1);
  };

  const nextQuiz = () => {
    setQuizAnswer('');
    setQuizResult(null);
    if (quizIndex < quizOrder.length - 1) {
      setQuizIndex(prev => prev + 1);
    } else {
      const order = Array.from({ length: words.length }, (_, i) => i);
      for (let i = order.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [order[i], order[j]] = [order[j], order[i]];
      }
      setQuizOrder(order);
      setQuizIndex(0);
    }
  };

  const checkWriteAnswer = () => {
    if (!writeInput.trim() || writeOrder.length === 0) return;
    const currentWord = words[writeOrder[writeIndex]];
    if (!currentWord) return;
    const isCorrect = writeInput.trim() === currentWord.chinese;
    setWriteResult(isCorrect ? 'correct' : 'wrong');
    setWriteTotal(prev => prev + 1);
    if (isCorrect) setWriteScore(prev => prev + 1);
  };

  const nextWrite = () => {
    setWriteInput('');
    setWriteResult(null);
    setShowHint(false);
    if (writeIndex < writeOrder.length - 1) {
      setWriteIndex(prev => prev + 1);
    } else {
      const order = Array.from({ length: words.length }, (_, i) => i);
      for (let i = order.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [order[i], order[j]] = [order[j], order[i]];
      }
      setWriteOrder(order);
      setWriteIndex(0);
    }
  };

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (modeRef.current === 'cards') {
        if (e.key === 'ArrowRight') swipeRight();
        if (e.key === 'ArrowLeft') swipeLeft();
        if (e.key === ' ') {
          e.preventDefault();
          setIsFlipped(!isFlippedRef.current);
        }
      }
    };
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [swipeRight, swipeLeft]);

  return (
    <div className="min-h-screen bg-gradient-to-br from-red-50 via-orange-50 to-yellow-50">
      {/* Header */}
      <header className="bg-white/80 backdrop-blur-sm shadow-sm border-b border-red-100 sticky top-0 z-10">
        <div className="max-w-4xl mx-auto px-4 py-4">
          <div className="flex flex-col sm:flex-row items-center justify-between gap-3">
            <h1 className="text-2xl font-bold text-red-700 flex items-center gap-2">
              <span className="text-3xl">🀄</span>
              <span>汉字卡片</span>
            </h1>
            <div className="flex flex-wrap gap-1 bg-gray-100 rounded-xl p-1">
              <button
                onClick={() => setMode('add')}
                className={`px-4 py-2 rounded-lg text-sm font-medium transition-all ${
                  mode === 'add' ? 'bg-white shadow text-red-700' : 'text-gray-600 hover:text-red-600'
                }`}
              >
                ✏️ Добавить
              </button>
              <button
                onClick={() => { setMode('cards'); startCardsSession(); }}
                className={`px-4 py-2 rounded-lg text-sm font-medium transition-all ${
                  mode === 'cards' ? 'bg-white shadow text-red-700' : 'text-gray-600 hover:text-red-600'
                }`}
              >
                🃏 Карточки
              </button>
              <button
                onClick={() => { setMode('quiz'); setQuizScore(0); setQuizTotal(0); }}
                className={`px-4 py-2 rounded-lg text-sm font-medium transition-all ${
                  mode === 'quiz' ? 'bg-white shadow text-red-700' : 'text-gray-600 hover:text-red-600'
                }`}
              >
                🧠 Тест
              </button>
              <button
                onClick={() => { setMode('write'); setWriteScore(0); setWriteTotal(0); }}
                className={`px-4 py-2 rounded-lg text-sm font-medium transition-all ${
                  mode === 'write' ? 'bg-white shadow text-red-700' : 'text-gray-600 hover:text-red-600'
                }`}
              >
                ✍️ Написать
              </button>
            </div>
          </div>
        </div>
      </header>

      <main className="max-w-4xl mx-auto px-4 py-8">
        {/* Add Words Mode */}
        {mode === 'add' && (
          <div className="space-y-6">
            <div className="bg-white rounded-2xl shadow-lg p-6 border border-red-100">
              <h2 className="text-xl font-semibold text-gray-800 mb-4">Добавить слово</h2>
              <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                <div>
                  <label className="block text-sm font-medium text-gray-600 mb-1">Китайский иероглиф</label>
                  <input
                    type="text"
                    value={chinese}
                    onChange={(e) => setChinese(e.target.value)}
                    placeholder="例如: 你好"
                    className="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none text-lg transition-all"
                    onKeyDown={(e) => e.key === 'Enter' && addWord()}
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-600 mb-1">Пиньинь</label>
                  <input
                    type="text"
                    value={pinyin}
                    onChange={(e) => setPinyin(e.target.value)}
                    placeholder="nǐ hǎo"
                    className="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none transition-all"
                    onKeyDown={(e) => e.key === 'Enter' && addWord()}
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-600 mb-1">Перевод</label>
                  <input
                    type="text"
                    value={translation}
                    onChange={(e) => setTranslation(e.target.value)}
                    placeholder="привет"
                    className="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none transition-all"
                    onKeyDown={(e) => e.key === 'Enter' && addWord()}
                  />
                </div>
              </div>
              <button
                onClick={addWord}
                disabled={!chinese.trim() || !translation.trim()}
                className="mt-4 px-6 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-all shadow-md hover:shadow-lg"
              >
                Добавить карточку
              </button>
            </div>

            {/* Bulk Add */}
            <div className="bg-white rounded-2xl shadow-lg p-6 border border-red-100">
              <h2 className="text-xl font-semibold text-gray-800 mb-2">Массовый ввод</h2>
              <p className="text-sm text-gray-500 mb-4">
                Вставьте список слов. Каждая строка — одно слово. Формат: <code className="bg-gray-100 px-1 rounded">иероглиф - пиньинь - перевод</code> или <code className="bg-gray-100 px-1 rounded">иероглиф - перевод</code>
              </p>
              <textarea
                value={bulkText}
                onChange={(e) => handleBulkChange(e.target.value)}
                placeholder={`你好 - nǐ hǎo - привет\n谢谢 - xiè xie - спасибо\n再见 - zài jiàn - до свидания\n学习 - xué xí - учиться`}
                rows={8}
                className="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none transition-all font-mono text-sm resize-y"
              />

              {bulkPreview.length > 0 && (
                <div className="mt-4">
                  <p className="text-sm font-medium text-gray-600 mb-2">
                    Будет добавлено: {bulkPreview.length} слов
                  </p>
                  <div className="max-h-40 overflow-y-auto space-y-1 mb-4">
                    {bulkPreview.map((w, i) => (
                      <div key={i} className="flex items-center gap-3 text-sm p-2 bg-red-50 rounded-lg">
                        <span className="font-bold text-red-700">{w.chinese}</span>
                        {w.pinyin && <span className="text-gray-500 italic">{w.pinyin}</span>}
                        <span className="text-gray-700">{w.translation}</span>
                      </div>
                    ))}
                  </div>
                </div>
              )}

              <button
                onClick={addBulkWords}
                disabled={bulkPreview.length === 0}
                className="px-6 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-all shadow-md hover:shadow-lg"
              >
                Добавить все ({bulkPreview.length})
              </button>
            </div>

            {/* Word List */}
            {words.length > 0 && (
              <div className="bg-white rounded-2xl shadow-lg p-6 border border-red-100">
                <h2 className="text-xl font-semibold text-gray-800 mb-4">
                  Ваши слова ({words.length})
                </h2>
                <div className="space-y-2">
                  {words.map((word) => (
                    <div
                      key={word.id}
                      className="flex items-center justify-between p-4 bg-gray-50 rounded-xl hover:bg-red-50 transition-all group"
                    >
                      <div className="flex items-center gap-4 flex-wrap">
                        <span className="text-2xl font-bold text-red-700">{word.chinese}</span>
                        {word.pinyin && (
                          <span className="text-sm text-gray-500 italic">{word.pinyin}</span>
                        )}
                        <span className="text-gray-700">{word.translation}</span>
                      </div>
                      <button
                        onClick={() => deleteWord(word.id)}
                        className="opacity-0 group-hover:opacity-100 text-red-400 hover:text-red-600 transition-all p-2 text-lg"
                      >
                        ✕
                      </button>
                    </div>
                  ))}
                </div>
              </div>
            )}

            {words.length === 0 && (
              <div className="text-center py-12 text-gray-400">
                <div className="text-6xl mb-4">📝</div>
                <p className="text-lg">Добавьте первое китайское слово для изучения!</p>
              </div>
            )}
          </div>
        )}

        {/* Cards Mode */}
        {mode === 'cards' && (() => {
          const isFinished = cardIndex >= cardQueue.length;
          const currentCardWord = !isFinished && cardQueue.length > 0
            ? words[cardQueue[cardIndex]]
            : null;

          return (
            <div className="space-y-6">
              {words.length === 0 ? (
                <div className="text-center py-12 text-gray-400">
                  <div className="text-6xl mb-4">🃏</div>
                  <p className="text-lg">Сначала добавьте слова во вкладке "Добавить"</p>
                </div>
              ) : isFinished ? (
                /* Finished screen */
                <div className="bg-white rounded-3xl shadow-xl border-2 border-red-100 p-8 text-center">
                  <div className="text-6xl mb-4">🎉</div>
                  <h2 className="text-2xl font-bold text-gray-800 mb-2">Раунд завершён!</h2>
                  <p className="text-gray-500 mb-6">Все карточки пройдены</p>

                  <div className="flex justify-center gap-8 mb-8">
                    <div className="text-center">
                      <div className="text-4xl font-bold text-green-600">{knownCount}</div>
                      <div className="text-sm text-gray-500 mt-1">Помню ✓</div>
                    </div>
                    <div className="text-center">
                      <div className="text-4xl font-bold text-orange-500">{unknownCount}</div>
                      <div className="text-sm text-gray-500 mt-1">Повторить ↩</div>
                    </div>
                  </div>

                  <div className="w-full bg-gray-200 rounded-full h-3 mb-6">
                    <div
                      className="bg-green-500 h-3 rounded-full transition-all"
                      style={{ width: `${((knownCount) / (knownCount + unknownCount)) * 100}%` }}
                    ></div>
                  </div>

                  <button
                    onClick={startCardsSession}
                    className="px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 transition-all shadow-md"
                  >
                    Начать заново 🔄
                  </button>
                </div>
              ) : currentCardWord ? (
                <>
                  {/* Progress */}
                  <div className="flex justify-between items-center">
                    <span className="text-gray-500 text-sm">
                      Осталось: {cardQueue.length - cardIndex}
                    </span>
                    <div className="flex gap-3 text-sm">
                      <span className="text-green-600 font-medium">✓ {knownCount}</span>
                      <span className="text-orange-500 font-medium">↩ {unknownCount}</span>
                    </div>
                  </div>

                  {/* Progress bar */}
                  <div className="w-full bg-gray-200 rounded-full h-2">
                    <div
                      className="bg-red-500 h-2 rounded-full transition-all duration-300"
                      style={{ width: `${(cardIndex / cardQueue.length) * 100}%` }}
                    ></div>
                  </div>

                  <label className="flex items-center gap-2 text-sm text-gray-600 cursor-pointer">
                    <input
                      type="checkbox"
                      checked={showPinyin}
                      onChange={(e) => setShowPinyin(e.target.checked)}
                      className="w-4 h-4 rounded border-gray-300 text-red-600 focus:ring-red-500"
                    />
                    Показывать пиньинь
                  </label>

                  {/* Flashcard with swipe */}
                  <div
                    className="relative"
                    onTouchStart={(e) => setTouchStart(e.touches[0].clientX)}
                    onTouchMove={(e) => {
                      if (touchStart !== null) {
                        setTouchDelta(e.touches[0].clientX - touchStart);
                      }
                    }}
                    onTouchEnd={() => {
                      if (touchDelta > 80) {
                        swipeRight();
                      } else if (touchDelta < -80) {
                        swipeLeft();
                      }
                      setTouchStart(null);
                      setTouchDelta(0);
                    }}
                    onClick={() => setIsFlipped(!isFlipped)}
                    style={{ perspective: '1000px' }}
                  >
                    <div
                      className="relative w-full h-80 transition-all duration-300 cursor-pointer"
                      style={{
                        transformStyle: 'preserve-3d',
                        transform: `
                          ${isFlipped ? 'rotateY(180deg)' : 'rotateY(0deg)'}
                          ${swipeDirection === 'left' ? 'translateX(-150%) rotate(-15deg)' : ''}
                          ${swipeDirection === 'right' ? 'translateX(150%) rotate(15deg)' : ''}
                          ${!swipeDirection && touchDelta !== 0 ? `translateX(${touchDelta}px) rotate(${touchDelta * 0.05}deg)` : ''}
                        `,
                        transition: swipeDirection ? 'transform 0.3s ease-out' : (touchDelta !== 0 ? 'none' : 'transform 0.5s'),
                        opacity: swipeDirection ? 0 : 1,
                      }}
                    >
                      {/* Front - перевод */}
                      <div
                        className="absolute inset-0 bg-white rounded-3xl shadow-xl border-2 border-red-100 flex flex-col items-center justify-center p-8"
                        style={{ backfaceVisibility: 'hidden' }}
                      >
                        <div className="text-3xl font-bold text-gray-800 text-center mb-4">
                          {currentCardWord.translation}
                        </div>
                        <div className="absolute bottom-4 text-sm text-gray-400">
                          Нажмите, чтобы увидеть иероглиф
                        </div>
                      </div>
                      {/* Back - иероглиф */}
                      <div
                        className="absolute inset-0 bg-gradient-to-br from-red-500 to-red-700 rounded-3xl shadow-xl flex flex-col items-center justify-center p-8"
                        style={{ backfaceVisibility: 'hidden', transform: 'rotateY(180deg)' }}
                      >
                        <div className="text-6xl font-bold text-white mb-4">
                          {currentCardWord.chinese}
                        </div>
                        {showPinyin && currentCardWord.pinyin && (
                          <div className="text-xl text-red-100 italic">
                            {currentCardWord.pinyin}
                          </div>
                        )}
                        <div className="absolute bottom-4 text-sm text-red-200">
                          Нажмите, чтобы вернуться
                        </div>
                      </div>
                    </div>

                    {/* Swipe indicators */}
                    {touchDelta > 30 && (
                      <div className="absolute top-4 right-4 bg-green-500 text-white px-4 py-2 rounded-xl font-bold text-lg shadow-lg animate-pulse">
                        ✓ Помню
                      </div>
                    )}
                    {touchDelta < -30 && (
                      <div className="absolute top-4 left-4 bg-orange-500 text-white px-4 py-2 rounded-xl font-bold text-lg shadow-lg animate-pulse">
                        ↩ Не помню
                      </div>
                    )}
                  </div>

                  {/* Swipe buttons */}
                  <div className="flex justify-center gap-6">
                    <button
                      onClick={(e) => { e.stopPropagation(); swipeLeft(); }}
                      className="flex flex-col items-center gap-1 px-6 py-4 bg-white rounded-2xl shadow-md hover:shadow-lg transition-all border-2 border-orange-200 hover:border-orange-400 active:scale-95"
                    >
                      <span className="text-3xl">↩️</span>
                      <span className="text-sm font-medium text-orange-600">Не помню</span>
                    </button>
                    <button
                      onClick={(e) => { e.stopPropagation(); setIsFlipped(!isFlipped); }}
                      className="flex flex-col items-center gap-1 px-6 py-4 bg-red-600 text-white rounded-2xl shadow-md hover:shadow-lg transition-all hover:bg-red-700 active:scale-95"
                    >
                      <span className="text-3xl">🔄</span>
                      <span className="text-sm font-medium">Перевернуть</span>
                    </button>
                    <button
                      onClick={(e) => { e.stopPropagation(); swipeRight(); }}
                      className="flex flex-col items-center gap-1 px-6 py-4 bg-white rounded-2xl shadow-md hover:shadow-lg transition-all border-2 border-green-200 hover:border-green-400 active:scale-95"
                    >
                      <span className="text-3xl">✓</span>
                      <span className="text-sm font-medium text-green-600">Помню</span>
                    </button>
                  </div>

                  <p className="text-center text-sm text-gray-400">
                    💡 Свайпните карточку или используйте кнопки. ← → на клавиатуре
                  </p>
                </>
              ) : null}
            </div>
          );
        })()}

        {/* Quiz Mode */}
        {mode === 'quiz' && (() => {
          const currentWord = quizOrder.length > 0 && quizIndex < quizOrder.length
            ? words[quizOrder[quizIndex]]
            : null;

          return (
            <div className="space-y-6">
              {words.length === 0 || !currentWord ? (
                <div className="text-center py-12 text-gray-400">
                  <div className="text-6xl mb-4">🧠</div>
                  <p className="text-lg">
                    {words.length === 0
                      ? 'Сначала добавьте слова во вкладке "Добавить"'
                      : 'Загрузка...'}
                  </p>
                </div>
              ) : (
                <>
                  <div className="flex justify-between items-center">
                    <span className="text-gray-500">
                      Вопрос {quizIndex + 1} из {quizOrder.length}
                    </span>
                    <span className="text-sm font-medium text-gray-600 bg-gray-100 px-3 py-1 rounded-full">
                      Счёт: {quizScore}/{quizTotal}
                    </span>
                  </div>

                  <div className="bg-white rounded-3xl shadow-xl border-2 border-red-100 p-8 text-center">
                    <p className="text-gray-500 mb-2">Что означает:</p>
                    <div className="text-6xl font-bold text-red-700 mb-2">
                      {currentWord.chinese}
                    </div>
                    {currentWord.pinyin && (
                      <div className="text-xl text-gray-400 italic mb-6">
                        {currentWord.pinyin}
                      </div>
                    )}

                    <div className="max-w-md mx-auto mt-6">
                      <input
                        type="text"
                        value={quizAnswer}
                        onChange={(e) => setQuizAnswer(e.target.value)}
                        onKeyDown={(e) => {
                          if (e.key === 'Enter') {
                            if (quizResult) nextQuiz();
                            else checkQuizAnswer();
                          }
                        }}
                        placeholder="Введите перевод..."
                        disabled={quizResult !== null}
                        className="w-full px-6 py-4 rounded-xl border-2 border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none text-lg text-center transition-all disabled:bg-gray-50"
                        autoFocus
                      />
                    </div>

                    {quizResult && (
                      <div className={`mt-4 p-4 rounded-xl ${
                        quizResult === 'correct' ? 'bg-green-50 text-green-700 border border-green-200' : 'bg-red-50 text-red-700 border border-red-200'
                      }`}>
                        {quizResult === 'correct' ? (
                          <p className="font-medium">✅ Правильно!</p>
                        ) : (
                          <p className="font-medium">
                            ❌ Неправильно. Правильный ответ: <strong>{currentWord.translation}</strong>
                          </p>
                        )}
                      </div>
                    )}

                    {!quizResult ? (
                      <button
                        onClick={checkQuizAnswer}
                        disabled={!quizAnswer.trim()}
                        className="mt-4 px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-all shadow-md"
                      >
                        Проверить
                      </button>
                    ) : (
                      <button
                        onClick={nextQuiz}
                        className="mt-4 px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 transition-all shadow-md"
                      >
                        {quizIndex < quizOrder.length - 1 ? 'Следующий вопрос →' : 'Начать заново 🔄'}
                      </button>
                    )}
                  </div>

                  {/* Progress bar */}
                  <div className="w-full bg-gray-200 rounded-full h-2">
                    <div
                      className="bg-red-500 h-2 rounded-full transition-all duration-300"
                      style={{ width: `${((quizIndex + 1) / quizOrder.length) * 100}%` }}
                    ></div>
                  </div>
                </>
              )}
            </div>
          );
        })()}

        {/* Write Mode */}
        {mode === 'write' && (() => {
          const currentWord = writeOrder.length > 0 && writeIndex < writeOrder.length
            ? words[writeOrder[writeIndex]]
            : null;

          return (
            <div className="space-y-6">
              {words.length === 0 || !currentWord ? (
                <div className="text-center py-12 text-gray-400">
                  <div className="text-6xl mb-4">✍️</div>
                  <p className="text-lg">
                    {words.length === 0
                      ? 'Сначала добавьте слова во вкладке "Добавить"'
                      : 'Загрузка...'}
                  </p>
                </div>
              ) : (
                <>
                  <div className="flex justify-between items-center">
                    <span className="text-gray-500">
                      Слово {writeIndex + 1} из {writeOrder.length}
                    </span>
                    <span className="text-sm font-medium text-gray-600 bg-gray-100 px-3 py-1 rounded-full">
                      Счёт: {writeScore}/{writeTotal}
                    </span>
                  </div>

                  <div className="bg-white rounded-3xl shadow-xl border-2 border-red-100 p-8 text-center">
                    <p className="text-gray-500 mb-2">Напишите иероглиф:</p>
                    <div className="text-4xl font-bold text-red-700 mb-6">
                      {currentWord.translation}
                    </div>

                    <div className="max-w-md mx-auto">
                      <input
                        type="text"
                        value={writeInput}
                        onChange={(e) => setWriteInput(e.target.value)}
                        onKeyDown={(e) => {
                          if (e.key === 'Enter') {
                            if (writeResult) nextWrite();
                            else checkWriteAnswer();
                          }
                        }}
                        placeholder="Введите иероглиф..."
                        disabled={writeResult !== null}
                        className="w-full px-6 py-4 rounded-xl border-2 border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none text-3xl text-center transition-all disabled:bg-gray-50"
                        autoFocus
                      />
                    </div>

                    {/* Hint button */}
                    {currentWord.pinyin && !writeResult && (
                      <div className="mt-4">
                        {!showHint ? (
                          <button
                            onClick={() => setShowHint(true)}
                            className="px-4 py-2 text-sm text-gray-500 hover:text-red-600 transition-all underline"
                          >
                            💡 Подсказка (пиньинь)
                          </button>
                        ) : (
                          <div className="inline-block px-4 py-2 bg-yellow-50 border border-yellow-200 rounded-xl">
                            <span className="text-lg text-yellow-700 italic">
                              {currentWord.pinyin}
                            </span>
                          </div>
                        )}
                      </div>
                    )}

                    {writeResult && (
                      <div className={`mt-4 p-4 rounded-xl ${
                        writeResult === 'correct' ? 'bg-green-50 text-green-700 border border-green-200' : 'bg-red-50 text-red-700 border border-red-200'
                      }`}>
                        {writeResult === 'correct' ? (
                          <p className="font-medium">✅ Правильно!</p>
                        ) : (
                          <p className="font-medium">
                            ❌ Неправильно. Правильный ответ: <strong className="text-2xl">{currentWord.chinese}</strong>
                            {currentWord.pinyin && (
                              <span className="block text-sm mt-1 text-gray-600 italic">
                                {currentWord.pinyin}
                              </span>
                            )}
                          </p>
                        )}
                      </div>
                    )}

                    {!writeResult ? (
                      <button
                        onClick={checkWriteAnswer}
                        disabled={!writeInput.trim()}
                        className="mt-4 px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-all shadow-md"
                      >
                        Проверить
                      </button>
                    ) : (
                      <button
                        onClick={nextWrite}
                        className="mt-4 px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 transition-all shadow-md"
                      >
                        {writeIndex < writeOrder.length - 1 ? 'Следующее слово →' : 'Начать заново 🔄'}
                      </button>
                    )}
                  </div>

                  {/* Progress bar */}
                  <div className="w-full bg-gray-200 rounded-full h-2">
                    <div
                      className="bg-red-500 h-2 rounded-full transition-all duration-300"
                      style={{ width: `${((writeIndex + 1) / writeOrder.length) * 100}%` }}
                    ></div>
                  </div>
                </>
              )}
            </div>
          );
        })()}
      </main>

      {/* Footer */}
      <footer className="text-center py-6 text-gray-400 text-sm">
        <p>汉字卡片 — Учите китайский с удовольствием 🎋</p>
      </footer>
    </div>
  );
}

export default App;


+++ src/App.tsx (修改后)
import { useState, useEffect, useCallback, useRef } from 'react';

interface Word {
  id: string;
  chinese: string;
  pinyin: string;
  translation: string;
}

type Mode = 'add' | 'cards' | 'quiz' | 'write';

function App() {
  const [words, setWords] = useState<Word[]>(() => {
    const saved = localStorage.getItem('chinese-words');
    return saved ? JSON.parse(saved) : [];
  });
  const [mode, setMode] = useState<Mode>('add');

  // Add word form
  const [chinese, setChinese] = useState('');
  const [pinyin, setPinyin] = useState('');
  const [translation, setTranslation] = useState('');

  // Bulk add
  const [bulkText, setBulkText] = useState('');
  const [bulkPreview, setBulkPreview] = useState<Word[]>([]);
  const [showExport, setShowExport] = useState(false);
  const [copied, setCopied] = useState(false);

  // Cards mode
  const [cardQueue, setCardQueue] = useState<number[]>([]);
  const [cardIndex, setCardIndex] = useState(0);
  const [isFlipped, setIsFlipped] = useState(false);
  const [showPinyin, setShowPinyin] = useState(true);
  const [knownCount, setKnownCount] = useState(0);
  const [unknownCount, setUnknownCount] = useState(0);
  const [swipeDirection, setSwipeDirection] = useState<'left' | 'right' | null>(null);
  const [touchStart, setTouchStart] = useState<number | null>(null);
  const [touchDelta, setTouchDelta] = useState(0);

  // Quiz mode
  const [quizAnswer, setQuizAnswer] = useState('');
  const [quizResult, setQuizResult] = useState<'correct' | 'wrong' | null>(null);
  const [quizScore, setQuizScore] = useState(0);
  const [quizTotal, setQuizTotal] = useState(0);
  const [quizOrder, setQuizOrder] = useState<number[]>([]);
  const [quizIndex, setQuizIndex] = useState(0);

  // Write mode
  const [writeInput, setWriteInput] = useState('');
  const [writeResult, setWriteResult] = useState<'correct' | 'wrong' | null>(null);
  const [showHint, setShowHint] = useState(false);
  const [writeOrder, setWriteOrder] = useState<number[]>([]);
  const [writeIndex, setWriteIndex] = useState(0);
  const [writeScore, setWriteScore] = useState(0);
  const [writeTotal, setWriteTotal] = useState(0);

  const modeRef = useRef(mode);
  const wordsLenRef = useRef(words.length);
  const isFlippedRef = useRef(isFlipped);

  useEffect(() => { modeRef.current = mode; }, [mode]);
  useEffect(() => { wordsLenRef.current = words.length; }, [words.length]);
  useEffect(() => { isFlippedRef.current = isFlipped; }, [isFlipped]);

  useEffect(() => {
    localStorage.setItem('chinese-words', JSON.stringify(words));
  }, [words]);

  useEffect(() => {
    if (mode === 'quiz' && words.length > 0) {
      const order = Array.from({ length: words.length }, (_, i) => i);
      for (let i = order.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [order[i], order[j]] = [order[j], order[i]];
      }
      setQuizOrder(order);
      setQuizIndex(0);
      setQuizAnswer('');
      setQuizResult(null);
    }
    if (mode === 'write' && words.length > 0) {
      const order = Array.from({ length: words.length }, (_, i) => i);
      for (let i = order.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [order[i], order[j]] = [order[j], order[i]];
      }
      setWriteOrder(order);
      setWriteIndex(0);
      setWriteInput('');
      setWriteResult(null);
      setShowHint(false);
    }
    if (mode === 'cards' && words.length > 0 && cardQueue.length === 0) {
      startCardsSession();
    }
  }, [mode, words.length]);

  const addWord = () => {
    if (!chinese.trim() || !translation.trim()) return;
    const newWord: Word = {
      id: Date.now().toString(),
      chinese: chinese.trim(),
      pinyin: pinyin.trim(),
      translation: translation.trim(),
    };
    setWords([...words, newWord]);
    setChinese('');
    setPinyin('');
    setTranslation('');
  };

  const deleteWord = (id: string) => {
    setWords(words.filter(w => w.id !== id));
  };

  // Parse bulk text into words
  const parseBulkText = (text: string): Word[] => {
    const lines = text.split('\n').filter(line => line.trim());
    const parsed: Word[] = [];

    lines.forEach((line, idx) => {
      // Try different separators: tab, " - ", " = ", " → ", " → ", ","
      let parts: string[] = [];

      if (line.includes('\t')) {
        parts = line.split('\t').map(s => s.trim()).filter(Boolean);
      } else if (line.includes(' - ')) {
        parts = line.split(' - ').map(s => s.trim()).filter(Boolean);
      } else if (line.includes(' = ')) {
        parts = line.split(' = ').map(s => s.trim()).filter(Boolean);
      } else if (line.includes(' → ')) {
        parts = line.split(' → ').map(s => s.trim()).filter(Boolean);
      } else if (line.includes('->')) {
        parts = line.split('->').map(s => s.trim()).filter(Boolean);
      } else if (line.includes(',')) {
        parts = line.split(',').map(s => s.trim()).filter(Boolean);
      } else {
        // Try splitting by spaces — first word is Chinese, rest is translation
        const trimmed = line.trim();
        const spaceIdx = trimmed.indexOf(' ');
        if (spaceIdx > 0) {
          parts = [trimmed.slice(0, spaceIdx), trimmed.slice(spaceIdx + 1).trim()];
        } else {
          parts = [trimmed];
        }
      }

      if (parts.length >= 2) {
        parsed.push({
          id: `bulk-${Date.now()}-${idx}`,
          chinese: parts[0],
          pinyin: parts.length >= 3 ? parts[1] : '',
          translation: parts.length >= 3 ? parts[2] : parts[1],
        });
      }
    });

    return parsed;
  };

  const handleBulkChange = (text: string) => {
    setBulkText(text);
    setBulkPreview(parseBulkText(text));
  };

  const addBulkWords = () => {
    if (bulkPreview.length === 0) return;
    setWords([...words, ...bulkPreview]);
    setBulkText('');
    setBulkPreview([]);
  };

  const shuffleArray = (arr: number[]): number[] => {
    const result = [...arr];
    for (let i = result.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [result[i], result[j]] = [result[j], result[i]];
    }
    return result;
  };

  const startCardsSession = useCallback(() => {
    const indices = Array.from({ length: words.length }, (_, i) => i);
    setCardQueue(shuffleArray(indices));
    setCardIndex(0);
    setIsFlipped(false);
    setKnownCount(0);
    setUnknownCount(0);
    setSwipeDirection(null);
  }, [words.length]);

  const swipeRight = useCallback(() => {
    // Помню - убираем карточку
    setSwipeDirection('right');
    setTimeout(() => {
      setKnownCount(prev => prev + 1);
      setCardIndex(prev => prev + 1);
      setIsFlipped(false);
      setSwipeDirection(null);
    }, 300);
  }, []);

  const swipeLeft = useCallback(() => {
    // Не помню - возвращаем карточку в конец очереди
    setSwipeDirection('left');
    setTimeout(() => {
      setUnknownCount(prev => prev + 1);
      const currentCardWordIndex = cardQueue[cardIndex];
      // Удаляем текущую карточку и добавляем в конец
      const newQueue = [...cardQueue.slice(0, cardIndex), ...cardQueue.slice(cardIndex + 1), currentCardWordIndex];
      setCardQueue(newQueue);
      // Индекс остаётся тем же, т.к. мы удалили элемент перед ним
      setIsFlipped(false);
      setSwipeDirection(null);
    }, 300);
  }, [cardQueue, cardIndex]);

  const checkQuizAnswer = () => {
    if (!quizAnswer.trim() || quizOrder.length === 0) return;
    const currentWord = words[quizOrder[quizIndex]];
    if (!currentWord) return;
    const isCorrect = quizAnswer.trim().toLowerCase() === currentWord.translation.toLowerCase();
    setQuizResult(isCorrect ? 'correct' : 'wrong');
    setQuizTotal(prev => prev + 1);
    if (isCorrect) setQuizScore(prev => prev + 1);
  };

  const nextQuiz = () => {
    setQuizAnswer('');
    setQuizResult(null);
    if (quizIndex < quizOrder.length - 1) {
      setQuizIndex(prev => prev + 1);
    } else {
      const order = Array.from({ length: words.length }, (_, i) => i);
      for (let i = order.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [order[i], order[j]] = [order[j], order[i]];
      }
      setQuizOrder(order);
      setQuizIndex(0);
    }
  };

  const checkWriteAnswer = () => {
    if (!writeInput.trim() || writeOrder.length === 0) return;
    const currentWord = words[writeOrder[writeIndex]];
    if (!currentWord) return;
    const isCorrect = writeInput.trim() === currentWord.chinese;
    setWriteResult(isCorrect ? 'correct' : 'wrong');
    setWriteTotal(prev => prev + 1);
    if (isCorrect) setWriteScore(prev => prev + 1);
  };

  const nextWrite = () => {
    setWriteInput('');
    setWriteResult(null);
    setShowHint(false);
    if (writeIndex < writeOrder.length - 1) {
      setWriteIndex(prev => prev + 1);
    } else {
      const order = Array.from({ length: words.length }, (_, i) => i);
      for (let i = order.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [order[i], order[j]] = [order[j], order[i]];
      }
      setWriteOrder(order);
      setWriteIndex(0);
    }
  };

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (modeRef.current === 'cards') {
        if (e.key === 'ArrowRight') swipeRight();
        if (e.key === 'ArrowLeft') swipeLeft();
        if (e.key === ' ') {
          e.preventDefault();
          setIsFlipped(!isFlippedRef.current);
        }
      }
    };
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [swipeRight, swipeLeft]);

  return (
    <div className="min-h-screen bg-gradient-to-br from-red-50 via-orange-50 to-yellow-50">
      {/* Header */}
      <header className="bg-white/80 backdrop-blur-sm shadow-sm border-b border-red-100 sticky top-0 z-10">
        <div className="max-w-4xl mx-auto px-4 py-4">
          <div className="flex flex-col sm:flex-row items-center justify-between gap-3">
            <h1 className="text-2xl font-bold text-red-700 flex items-center gap-2">
              <span className="text-3xl">🀄</span>
              <span>汉字卡片</span>
            </h1>
            <div className="flex flex-wrap gap-1 bg-gray-100 rounded-xl p-1">
              <button
                onClick={() => setMode('add')}
                className={`px-4 py-2 rounded-lg text-sm font-medium transition-all ${
                  mode === 'add' ? 'bg-white shadow text-red-700' : 'text-gray-600 hover:text-red-600'
                }`}
              >
                ✏️ Добавить
              </button>
              <button
                onClick={() => { setMode('cards'); startCardsSession(); }}
                className={`px-4 py-2 rounded-lg text-sm font-medium transition-all ${
                  mode === 'cards' ? 'bg-white shadow text-red-700' : 'text-gray-600 hover:text-red-600'
                }`}
              >
                🃏 Карточки
              </button>
              <button
                onClick={() => { setMode('quiz'); setQuizScore(0); setQuizTotal(0); }}
                className={`px-4 py-2 rounded-lg text-sm font-medium transition-all ${
                  mode === 'quiz' ? 'bg-white shadow text-red-700' : 'text-gray-600 hover:text-red-600'
                }`}
              >
                🧠 Тест
              </button>
              <button
                onClick={() => { setMode('write'); setWriteScore(0); setWriteTotal(0); }}
                className={`px-4 py-2 rounded-lg text-sm font-medium transition-all ${
                  mode === 'write' ? 'bg-white shadow text-red-700' : 'text-gray-600 hover:text-red-600'
                }`}
              >
                ✍️ Написать
              </button>
            </div>
          </div>
        </div>
      </header>

      <main className="max-w-4xl mx-auto px-4 py-8">
        {/* Add Words Mode */}
        {mode === 'add' && (
          <div className="space-y-6">
            <div className="bg-white rounded-2xl shadow-lg p-6 border border-red-100">
              <h2 className="text-xl font-semibold text-gray-800 mb-4">Добавить слово</h2>
              <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                <div>
                  <label className="block text-sm font-medium text-gray-600 mb-1">Китайский иероглиф</label>
                  <input
                    type="text"
                    value={chinese}
                    onChange={(e) => setChinese(e.target.value)}
                    placeholder="例如: 你好"
                    className="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none text-lg transition-all"
                    onKeyDown={(e) => e.key === 'Enter' && addWord()}
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-600 mb-1">Пиньинь</label>
                  <input
                    type="text"
                    value={pinyin}
                    onChange={(e) => setPinyin(e.target.value)}
                    placeholder="nǐ hǎo"
                    className="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none transition-all"
                    onKeyDown={(e) => e.key === 'Enter' && addWord()}
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-600 mb-1">Перевод</label>
                  <input
                    type="text"
                    value={translation}
                    onChange={(e) => setTranslation(e.target.value)}
                    placeholder="привет"
                    className="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none transition-all"
                    onKeyDown={(e) => e.key === 'Enter' && addWord()}
                  />
                </div>
              </div>
              <button
                onClick={addWord}
                disabled={!chinese.trim() || !translation.trim()}
                className="mt-4 px-6 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-all shadow-md hover:shadow-lg"
              >
                Добавить карточку
              </button>
            </div>

            {/* Bulk Add */}
            <div className="bg-white rounded-2xl shadow-lg p-6 border border-red-100">
              <h2 className="text-xl font-semibold text-gray-800 mb-2">Массовый ввод</h2>
              <p className="text-sm text-gray-500 mb-4">
                Вставьте список слов. Каждая строка — одно слово. Формат: <code className="bg-gray-100 px-1 rounded">иероглиф - пиньинь - перевод</code> или <code className="bg-gray-100 px-1 rounded">иероглиф - перевод</code>
              </p>
              <textarea
                value={bulkText}
                onChange={(e) => handleBulkChange(e.target.value)}
                placeholder={`你好 - nǐ hǎo - привет\n谢谢 - xiè xie - спасибо\n再见 - zài jiàn - до свидания\n学习 - xué xí - учиться`}
                rows={8}
                className="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none transition-all font-mono text-sm resize-y"
              />

              {bulkPreview.length > 0 && (
                <div className="mt-4">
                  <p className="text-sm font-medium text-gray-600 mb-2">
                    Будет добавлено: {bulkPreview.length} слов
                  </p>
                  <div className="max-h-40 overflow-y-auto space-y-1 mb-4">
                    {bulkPreview.map((w, i) => (
                      <div key={i} className="flex items-center gap-3 text-sm p-2 bg-red-50 rounded-lg">
                        <span className="font-bold text-red-700">{w.chinese}</span>
                        {w.pinyin && <span className="text-gray-500 italic">{w.pinyin}</span>}
                        <span className="text-gray-700">{w.translation}</span>
                      </div>
                    ))}
                  </div>
                </div>
              )}

              <button
                onClick={addBulkWords}
                disabled={bulkPreview.length === 0}
                className="px-6 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-all shadow-md hover:shadow-lg"
              >
                Добавить все ({bulkPreview.length})
              </button>
            </div>

            {/* Word List */}
            {words.length > 0 && (
              <div className="bg-white rounded-2xl shadow-lg p-6 border border-red-100">
                <div className="flex justify-between items-center mb-4">
                  <h2 className="text-xl font-semibold text-gray-800">
                    Ваши слова ({words.length})
                  </h2>
                  <button
                    onClick={() => setShowExport(!showExport)}
                    className="px-4 py-2 text-sm bg-gray-100 hover:bg-gray-200 rounded-lg transition-all flex items-center gap-2"
                  >
                    {showExport ? '🙈 Скрыть' : '📋 Экспорт'}
                  </button>
                </div>
                <div className="space-y-2">
                  {words.map((word) => (
                    <div
                      key={word.id}
                      className="flex items-center justify-between p-4 bg-gray-50 rounded-xl hover:bg-red-50 transition-all group"
                    >
                      <div className="flex items-center gap-4 flex-wrap">
                        <span className="text-2xl font-bold text-red-700">{word.chinese}</span>
                        {word.pinyin && (
                          <span className="text-sm text-gray-500 italic">{word.pinyin}</span>
                        )}
                        <span className="text-gray-700">{word.translation}</span>
                      </div>
                      <button
                        onClick={() => deleteWord(word.id)}
                        className="opacity-0 group-hover:opacity-100 text-red-400 hover:text-red-600 transition-all p-2 text-lg"
                      >
                        ✕
                      </button>
                    </div>
                  ))}
                </div>

                {/* Export section */}
                {showExport && (
                  <div className="mt-6 pt-6 border-t border-gray-200">
                    <div className="flex justify-between items-center mb-2">
                      <p className="text-sm font-medium text-gray-600">
                        Текст для импорта/бэкапа:
                      </p>
                      <button
                        onClick={() => {
                          const exportText = words
                            .map(w => w.pinyin ? `${w.chinese} - ${w.pinyin} - ${w.translation}` : `${w.chinese} - ${w.translation}`)
                            .join('\n');
                          navigator.clipboard.writeText(exportText);
                          setCopied(true);
                          setTimeout(() => setCopied(false), 2000);
                        }}
                        className="px-3 py-1 text-sm bg-red-600 text-white rounded-lg hover:bg-red-700 transition-all"
                      >
                        {copied ? '✓ Скопировано!' : '📋 Копировать'}
                      </button>
                    </div>
                    <textarea
                      readOnly
                      value={words
                        .map(w => w.pinyin ? `${w.chinese} - ${w.pinyin} - ${w.translation}` : `${w.chinese} - ${w.translation}`)
                        .join('\n')}
                      rows={8}
                      className="w-full px-4 py-3 rounded-xl border border-gray-200 bg-gray-50 font-mono text-sm resize-y"
                      onClick={(e) => (e.target as HTMLTextAreaElement).select()}
                    />
                    <p className="text-xs text-gray-400 mt-2">
                      💡 Скопируйте этот текст для бэкапа или переноса на другое устройство
                    </p>
                  </div>
                )}
              </div>
            )}

            {words.length === 0 && (
              <div className="text-center py-12 text-gray-400">
                <div className="text-6xl mb-4">📝</div>
                <p className="text-lg">Добавьте первое китайское слово для изучения!</p>
              </div>
            )}
          </div>
        )}

        {/* Cards Mode */}
        {mode === 'cards' && (() => {
          const isFinished = cardIndex >= cardQueue.length;
          const currentCardWord = !isFinished && cardQueue.length > 0
            ? words[cardQueue[cardIndex]]
            : null;

          return (
            <div className="space-y-6">
              {words.length === 0 ? (
                <div className="text-center py-12 text-gray-400">
                  <div className="text-6xl mb-4">🃏</div>
                  <p className="text-lg">Сначала добавьте слова во вкладке "Добавить"</p>
                </div>
              ) : isFinished ? (
                /* Finished screen */
                <div className="bg-white rounded-3xl shadow-xl border-2 border-red-100 p-8 text-center">
                  <div className="text-6xl mb-4">🎉</div>
                  <h2 className="text-2xl font-bold text-gray-800 mb-2">Раунд завершён!</h2>
                  <p className="text-gray-500 mb-6">Все карточки пройдены</p>

                  <div className="flex justify-center gap-8 mb-8">
                    <div className="text-center">
                      <div className="text-4xl font-bold text-green-600">{knownCount}</div>
                      <div className="text-sm text-gray-500 mt-1">Помню ✓</div>
                    </div>
                    <div className="text-center">
                      <div className="text-4xl font-bold text-orange-500">{unknownCount}</div>
                      <div className="text-sm text-gray-500 mt-1">Повторить ↩</div>
                    </div>
                  </div>

                  <div className="w-full bg-gray-200 rounded-full h-3 mb-6">
                    <div
                      className="bg-green-500 h-3 rounded-full transition-all"
                      style={{ width: `${((knownCount) / (knownCount + unknownCount)) * 100}%` }}
                    ></div>
                  </div>

                  <button
                    onClick={startCardsSession}
                    className="px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 transition-all shadow-md"
                  >
                    Начать заново 🔄
                  </button>
                </div>
              ) : currentCardWord ? (
                <>
                  {/* Progress */}
                  <div className="flex justify-between items-center">
                    <span className="text-gray-500 text-sm">
                      Осталось: {cardQueue.length - cardIndex}
                    </span>
                    <div className="flex gap-3 text-sm">
                      <span className="text-green-600 font-medium">✓ {knownCount}</span>
                      <span className="text-orange-500 font-medium">↩ {unknownCount}</span>
                    </div>
                  </div>

                  {/* Progress bar */}
                  <div className="w-full bg-gray-200 rounded-full h-2">
                    <div
                      className="bg-red-500 h-2 rounded-full transition-all duration-300"
                      style={{ width: `${(cardIndex / cardQueue.length) * 100}%` }}
                    ></div>
                  </div>

                  <label className="flex items-center gap-2 text-sm text-gray-600 cursor-pointer">
                    <input
                      type="checkbox"
                      checked={showPinyin}
                      onChange={(e) => setShowPinyin(e.target.checked)}
                      className="w-4 h-4 rounded border-gray-300 text-red-600 focus:ring-red-500"
                    />
                    Показывать пиньинь
                  </label>

                  {/* Flashcard with swipe */}
                  <div
                    className="relative"
                    onTouchStart={(e) => setTouchStart(e.touches[0].clientX)}
                    onTouchMove={(e) => {
                      if (touchStart !== null) {
                        setTouchDelta(e.touches[0].clientX - touchStart);
                      }
                    }}
                    onTouchEnd={() => {
                      if (touchDelta > 80) {
                        swipeRight();
                      } else if (touchDelta < -80) {
                        swipeLeft();
                      }
                      setTouchStart(null);
                      setTouchDelta(0);
                    }}
                    onClick={() => setIsFlipped(!isFlipped)}
                    style={{ perspective: '1000px' }}
                  >
                    <div
                      className="relative w-full h-80 transition-all duration-300 cursor-pointer"
                      style={{
                        transformStyle: 'preserve-3d',
                        transform: `
                          ${isFlipped ? 'rotateY(180deg)' : 'rotateY(0deg)'}
                          ${swipeDirection === 'left' ? 'translateX(-150%) rotate(-15deg)' : ''}
                          ${swipeDirection === 'right' ? 'translateX(150%) rotate(15deg)' : ''}
                          ${!swipeDirection && touchDelta !== 0 ? `translateX(${touchDelta}px) rotate(${touchDelta * 0.05}deg)` : ''}
                        `,
                        transition: swipeDirection ? 'transform 0.3s ease-out' : (touchDelta !== 0 ? 'none' : 'transform 0.5s'),
                        opacity: swipeDirection ? 0 : 1,
                      }}
                    >
                      {/* Front - перевод */}
                      <div
                        className="absolute inset-0 bg-white rounded-3xl shadow-xl border-2 border-red-100 flex flex-col items-center justify-center p-8"
                        style={{ backfaceVisibility: 'hidden' }}
                      >
                        <div className="text-3xl font-bold text-gray-800 text-center mb-4">
                          {currentCardWord.translation}
                        </div>
                        <div className="absolute bottom-4 text-sm text-gray-400">
                          Нажмите, чтобы увидеть иероглиф
                        </div>
                      </div>
                      {/* Back - иероглиф */}
                      <div
                        className="absolute inset-0 bg-gradient-to-br from-red-500 to-red-700 rounded-3xl shadow-xl flex flex-col items-center justify-center p-8"
                        style={{ backfaceVisibility: 'hidden', transform: 'rotateY(180deg)' }}
                      >
                        <div className="text-6xl font-bold text-white mb-4">
                          {currentCardWord.chinese}
                        </div>
                        {showPinyin && currentCardWord.pinyin && (
                          <div className="text-xl text-red-100 italic">
                            {currentCardWord.pinyin}
                          </div>
                        )}
                        <div className="absolute bottom-4 text-sm text-red-200">
                          Нажмите, чтобы вернуться
                        </div>
                      </div>
                    </div>

                    {/* Swipe indicators */}
                    {touchDelta > 30 && (
                      <div className="absolute top-4 right-4 bg-green-500 text-white px-4 py-2 rounded-xl font-bold text-lg shadow-lg animate-pulse">
                        ✓ Помню
                      </div>
                    )}
                    {touchDelta < -30 && (
                      <div className="absolute top-4 left-4 bg-orange-500 text-white px-4 py-2 rounded-xl font-bold text-lg shadow-lg animate-pulse">
                        ↩ Не помню
                      </div>
                    )}
                  </div>

                  {/* Swipe buttons */}
                  <div className="flex justify-center gap-6">
                    <button
                      onClick={(e) => { e.stopPropagation(); swipeLeft(); }}
                      className="flex flex-col items-center gap-1 px-6 py-4 bg-white rounded-2xl shadow-md hover:shadow-lg transition-all border-2 border-orange-200 hover:border-orange-400 active:scale-95"
                    >
                      <span className="text-3xl">↩️</span>
                      <span className="text-sm font-medium text-orange-600">Не помню</span>
                    </button>
                    <button
                      onClick={(e) => { e.stopPropagation(); setIsFlipped(!isFlipped); }}
                      className="flex flex-col items-center gap-1 px-6 py-4 bg-red-600 text-white rounded-2xl shadow-md hover:shadow-lg transition-all hover:bg-red-700 active:scale-95"
                    >
                      <span className="text-3xl">🔄</span>
                      <span className="text-sm font-medium">Перевернуть</span>
                    </button>
                    <button
                      onClick={(e) => { e.stopPropagation(); swipeRight(); }}
                      className="flex flex-col items-center gap-1 px-6 py-4 bg-white rounded-2xl shadow-md hover:shadow-lg transition-all border-2 border-green-200 hover:border-green-400 active:scale-95"
                    >
                      <span className="text-3xl">✓</span>
                      <span className="text-sm font-medium text-green-600">Помню</span>
                    </button>
                  </div>

                  <p className="text-center text-sm text-gray-400">
                    💡 Свайпните карточку или используйте кнопки. ← → на клавиатуре
                  </p>
                </>
              ) : null}
            </div>
          );
        })()}

        {/* Quiz Mode */}
        {mode === 'quiz' && (() => {
          const currentWord = quizOrder.length > 0 && quizIndex < quizOrder.length
            ? words[quizOrder[quizIndex]]
            : null;

          return (
            <div className="space-y-6">
              {words.length === 0 || !currentWord ? (
                <div className="text-center py-12 text-gray-400">
                  <div className="text-6xl mb-4">🧠</div>
                  <p className="text-lg">
                    {words.length === 0
                      ? 'Сначала добавьте слова во вкладке "Добавить"'
                      : 'Загрузка...'}
                  </p>
                </div>
              ) : (
                <>
                  <div className="flex justify-between items-center">
                    <span className="text-gray-500">
                      Вопрос {quizIndex + 1} из {quizOrder.length}
                    </span>
                    <span className="text-sm font-medium text-gray-600 bg-gray-100 px-3 py-1 rounded-full">
                      Счёт: {quizScore}/{quizTotal}
                    </span>
                  </div>

                  <div className="bg-white rounded-3xl shadow-xl border-2 border-red-100 p-8 text-center">
                    <p className="text-gray-500 mb-2">Что означает:</p>
                    <div className="text-6xl font-bold text-red-700 mb-2">
                      {currentWord.chinese}
                    </div>
                    {currentWord.pinyin && (
                      <div className="text-xl text-gray-400 italic mb-6">
                        {currentWord.pinyin}
                      </div>
                    )}

                    <div className="max-w-md mx-auto mt-6">
                      <input
                        type="text"
                        value={quizAnswer}
                        onChange={(e) => setQuizAnswer(e.target.value)}
                        onKeyDown={(e) => {
                          if (e.key === 'Enter') {
                            if (quizResult) nextQuiz();
                            else checkQuizAnswer();
                          }
                        }}
                        placeholder="Введите перевод..."
                        disabled={quizResult !== null}
                        className="w-full px-6 py-4 rounded-xl border-2 border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none text-lg text-center transition-all disabled:bg-gray-50"
                        autoFocus
                      />
                    </div>

                    {quizResult && (
                      <div className={`mt-4 p-4 rounded-xl ${
                        quizResult === 'correct' ? 'bg-green-50 text-green-700 border border-green-200' : 'bg-red-50 text-red-700 border border-red-200'
                      }`}>
                        {quizResult === 'correct' ? (
                          <p className="font-medium">✅ Правильно!</p>
                        ) : (
                          <p className="font-medium">
                            ❌ Неправильно. Правильный ответ: <strong>{currentWord.translation}</strong>
                          </p>
                        )}
                      </div>
                    )}

                    {!quizResult ? (
                      <button
                        onClick={checkQuizAnswer}
                        disabled={!quizAnswer.trim()}
                        className="mt-4 px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-all shadow-md"
                      >
                        Проверить
                      </button>
                    ) : (
                      <button
                        onClick={nextQuiz}
                        className="mt-4 px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 transition-all shadow-md"
                      >
                        {quizIndex < quizOrder.length - 1 ? 'Следующий вопрос →' : 'Начать заново 🔄'}
                      </button>
                    )}
                  </div>

                  {/* Progress bar */}
                  <div className="w-full bg-gray-200 rounded-full h-2">
                    <div
                      className="bg-red-500 h-2 rounded-full transition-all duration-300"
                      style={{ width: `${((quizIndex + 1) / quizOrder.length) * 100}%` }}
                    ></div>
                  </div>
                </>
              )}
            </div>
          );
        })()}

        {/* Write Mode */}
        {mode === 'write' && (() => {
          const currentWord = writeOrder.length > 0 && writeIndex < writeOrder.length
            ? words[writeOrder[writeIndex]]
            : null;

          return (
            <div className="space-y-6">
              {words.length === 0 || !currentWord ? (
                <div className="text-center py-12 text-gray-400">
                  <div className="text-6xl mb-4">✍️</div>
                  <p className="text-lg">
                    {words.length === 0
                      ? 'Сначала добавьте слова во вкладке "Добавить"'
                      : 'Загрузка...'}
                  </p>
                </div>
              ) : (
                <>
                  <div className="flex justify-between items-center">
                    <span className="text-gray-500">
                      Слово {writeIndex + 1} из {writeOrder.length}
                    </span>
                    <span className="text-sm font-medium text-gray-600 bg-gray-100 px-3 py-1 rounded-full">
                      Счёт: {writeScore}/{writeTotal}
                    </span>
                  </div>

                  <div className="bg-white rounded-3xl shadow-xl border-2 border-red-100 p-8 text-center">
                    <p className="text-gray-500 mb-2">Напишите иероглиф:</p>
                    <div className="text-4xl font-bold text-red-700 mb-6">
                      {currentWord.translation}
                    </div>

                    <div className="max-w-md mx-auto">
                      <input
                        type="text"
                        value={writeInput}
                        onChange={(e) => setWriteInput(e.target.value)}
                        onKeyDown={(e) => {
                          if (e.key === 'Enter') {
                            if (writeResult) nextWrite();
                            else checkWriteAnswer();
                          }
                        }}
                        placeholder="Введите иероглиф..."
                        disabled={writeResult !== null}
                        className="w-full px-6 py-4 rounded-xl border-2 border-gray-200 focus:border-red-400 focus:ring-2 focus:ring-red-100 outline-none text-3xl text-center transition-all disabled:bg-gray-50"
                        autoFocus
                      />
                    </div>

                    {/* Hint button */}
                    {currentWord.pinyin && !writeResult && (
                      <div className="mt-4">
                        {!showHint ? (
                          <button
                            onClick={() => setShowHint(true)}
                            className="px-4 py-2 text-sm text-gray-500 hover:text-red-600 transition-all underline"
                          >
                            💡 Подсказка (пиньинь)
                          </button>
                        ) : (
                          <div className="inline-block px-4 py-2 bg-yellow-50 border border-yellow-200 rounded-xl">
                            <span className="text-lg text-yellow-700 italic">
                              {currentWord.pinyin}
                            </span>
                          </div>
                        )}
                      </div>
                    )}

                    {writeResult && (
                      <div className={`mt-4 p-4 rounded-xl ${
                        writeResult === 'correct' ? 'bg-green-50 text-green-700 border border-green-200' : 'bg-red-50 text-red-700 border border-red-200'
                      }`}>
                        {writeResult === 'correct' ? (
                          <p className="font-medium">✅ Правильно!</p>
                        ) : (
                          <p className="font-medium">
                            ❌ Неправильно. Правильный ответ: <strong className="text-2xl">{currentWord.chinese}</strong>
                            {currentWord.pinyin && (
                              <span className="block text-sm mt-1 text-gray-600 italic">
                                {currentWord.pinyin}
                              </span>
                            )}
                          </p>
                        )}
                      </div>
                    )}

                    {!writeResult ? (
                      <button
                        onClick={checkWriteAnswer}
                        disabled={!writeInput.trim()}
                        className="mt-4 px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-all shadow-md"
                      >
                        Проверить
                      </button>
                    ) : (
                      <button
                        onClick={nextWrite}
                        className="mt-4 px-8 py-3 bg-red-600 text-white rounded-xl font-medium hover:bg-red-700 transition-all shadow-md"
                      >
                        {writeIndex < writeOrder.length - 1 ? 'Следующее слово →' : 'Начать заново 🔄'}
                      </button>
                    )}
                  </div>

                  {/* Progress bar */}
                  <div className="w-full bg-gray-200 rounded-full h-2">
                    <div
                      className="bg-red-500 h-2 rounded-full transition-all duration-300"
                      style={{ width: `${((writeIndex + 1) / writeOrder.length) * 100}%` }}
                    ></div>
                  </div>
                </>
              )}
            </div>
          );
        })()}
      </main>

      {/* Footer */}
      <footer className="text-center py-6 text-gray-400 text-sm">
        <p>汉字卡片 — Учите китайский с удовольствием 🎋</p>
      </footer>
    </div>
  );
}

export default App;
