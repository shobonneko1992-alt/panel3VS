/*
 * ╔═══════════════════════════════════════════════════════════╗
 * ║              PANEL BLAST — パネルでポン clone             ║
 * ║         React / Lucide-react • Single-file Artifact       ║
 * ╚═══════════════════════════════════════════════════════════╝
 */

import { useState, useEffect, useCallback, useRef } from "react";
import { RefreshCw, ArrowLeftRight, Trophy, Zap, Timer } from "lucide-react";

/* ═══════════════════════════════════════════════════════════════
   §1  CONSTANTS & PALETTE
   ═══════════════════════════════════════════════════════════════ */

const COLS        = 6;
const ROWS        = 12;
const CELL        = 46;       // px per block (board = 276 × 552)
const FLASH_MS    = 650;      // flash animation window
const CHAIN_DELAY    = 180;      // gap between chain-cycle steps
const STEP_MS        = 280;      // ms between each cell deletion in a match
const CELL_FLASH_MS  = 600;      // ms a cell flashes before being deleted
const START_ROWS       = 7;   // pre-filled rows on game start
const TIME_ATTACK_SECS = 60;  // 1分間タイムアタック
const RISE_INTERVAL_MS = 40;  // ティック間隔(ms) — 固定
// 上昇速度レベル: 1ティックあたりの上昇ピクセル数
const RISE_SPEEDS = [
  { label: 'OFF',  px: 0    },
  { label: 'LV 1', px: 0.25 },
  { label: 'LV 2', px: 0.5  },
  { label: 'LV 3', px: 1.0  },
  { label: 'LV 4', px: 2.0  },
  { label: 'LV 5', px: 3.5  },
];

// スコアアタック: ステージごとの目標スコア
const TARGET_SCORES = [500, 1500, 3500, 7000, 12000];

const COLORS = ["red", "blue", "green", "yellow", "purple", "teal"];

// Each color: gradient hi/mid/lo, text icon, glow tint
const PALETTE = {
  red:    { hi:"#FF8099", mid:"#FF2D55", lo:"#8B001A", icon:"◆", glow:"#FF2D55" },
  blue:   { hi:"#73C2FB", mid:"#0A84FF", lo:"#003380", icon:"●", glow:"#0A84FF" },
  green:  { hi:"#86EFAC", mid:"#30D158", lo:"#14532D", icon:"▲", glow:"#30D158" },
  yellow: { hi:"#FEF08A", mid:"#FFD60A", lo:"#78450B", icon:"★", glow:"#FFD60A" },
  purple: { hi:"#DDA0F9", mid:"#BF5AF2", lo:"#581C87", icon:"■", glow:"#BF5AF2" },
  teal:   { hi:"#BAE6FD", mid:"#5AC8FA", lo:"#0C4A6E", icon:"✦", glow:"#5AC8FA" },
  ojama:  { hi:"#9CA3AF", mid:"#4B5563", lo:"#1F2937", icon:"✕", glow:"#6B7280" },
  // ↑ お邪魔パネル: マッチには参加しない。隣接パネルが消えたとき一緒に消える
};

/* ═══════════════════════════════════════════════════════════════
   §2  PURE GAME LOGIC  (no React deps — fully testable)
   ═══════════════════════════════════════════════════════════════ */

const rnd = (n) => Math.floor(Math.random() * n);

/**
 * makeNextRow(prevRow)
 * Generates a single row of random blocks for the rising mechanic.
 * Avoids creating instant 3-matches with the row above it.
 */
function makeNextRow(prevRow) {
  return Array.from({ length: COLS }, (_, c) => {
    let color, attempts = 0;
    do {
      color = COLORS[rnd(COLORS.length)];
      attempts++;
    } while (
      attempts < 15 &&
      prevRow && prevRow[c]?.color === color
    );
    return { color };
  });
}

/**
 * buildGrid()
 * Creates a fresh 12×6 grid.
 * Fills the bottom START_ROWS rows with random blocks,
 * avoiding pre-existing 3-in-a-row matches.
 */
function buildGrid() {
  const g = Array.from({ length: ROWS }, () => Array(COLS).fill(null));

  for (let r = ROWS - START_ROWS; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      let color, attempts = 0;
      do {
        color = COLORS[rnd(COLORS.length)];
        attempts++;
      } while (
        attempts < 20 &&
        (
          // block horizontal match to the left
          (c >= 2 &&
            g[r][c - 1]?.color === color &&
            g[r][c - 2]?.color === color) ||
          // block vertical match above
          (r >= 2 &&
            g[r - 1][c]?.color === color &&
            g[r - 2][c]?.color === color)
        )
      );
      g[r][c] = { color };
    }
  }
  return g;
}

/**
 * findMatches(grid)
 * Returns a Set<"r,c"> of all cells that are part of a 3+ match
 * (horizontal or vertical). Does NOT include flashing/null cells.
 */
function findMatches(g) {
  const found = new Set();

  // ── Horizontal sweep ──
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; ) {
      const cell = g[r][c];
      if (!cell || cell.flashing) { c++; continue; }
      let n = 1;
      while (c + n < COLS && g[r][c + n]?.color === cell.color && !g[r][c + n].flashing) n++;
      if (n >= 3) for (let i = 0; i < n; i++) found.add(`${r},${c + i}`);
      c += n;
    }
  }

  // ── Vertical sweep ──
  for (let c = 0; c < COLS; c++) {
    for (let r = 0; r < ROWS; ) {
      const cell = g[r][c];
      if (!cell || cell.flashing) { r++; continue; }
      let n = 1;
      while (r + n < ROWS && g[r + n][c]?.color === cell.color && !g[r + n][c].flashing) n++;
      if (n >= 3) for (let i = 0; i < n; i++) found.add(`${r + i},${c}`);
      r += n;
    }
  }

  return found;
}

/**
 * applyGravity(grid)
 * Drops all blocks to the bottom of their column (no floating gaps).
 * Returns a new grid (immutable).
 */
function applyGravity(g) {
  const out = Array.from({ length: ROWS }, () => Array(COLS).fill(null));
  for (let c = 0; c < COLS; c++) {
    let w = ROWS - 1;
    for (let r = ROWS - 1; r >= 0; r--)
      if (g[r][c]) out[w--][c] = g[r][c];
  }
  return out;
}

/**
 * applyGravityStep(grid)
 * Drops every floating block exactly ONE row downward.
 * Returns { next: grid, moved: boolean }.
 * Repeated calls animate a slow fall one row at a time.
 */
function applyGravityStep(g) {
  let moved = false;
  const out = g.map(row => [...row]);
  for (let r = ROWS - 1; r > 0; r--) {
    for (let c = 0; c < COLS; c++) {
      if (!out[r][c] && out[r - 1][c]) {
        out[r][c]     = out[r - 1][c];
        out[r - 1][c] = null;
        moved = true;
      }
    }
  }
  return { next: out, moved };
}

/**
 * expandWithOjama(matches, grid)
 * マッチセルに隣接するお邪魔パネルをまとめて消去対象に追加する。
 */
function expandWithOjama(matches, g) {
  const dirs = [[-1,0],[1,0],[0,-1],[0,1]];
  const expanded = new Set(matches);
  for (const k of matches) {
    const [r, c] = k.split(',').map(Number);
    for (const [dr, dc] of dirs) {
      const nr = r + dr, nc = c + dc;
      if (nr >= 0 && nr < ROWS && nc >= 0 && nc < COLS)
        if (g[nr][nc]?.color === 'ojama') expanded.add(`${nr},${nc}`);
    }
  }
  return expanded;
}

/**
 * addOjamaToGrid(grid, count)
 * お邪魔パネルを count 個、上から押し込む（既存ブロックを上にシフト）。
 */
function addOjamaToGrid(g, count) {
  const ng = g.map(r => [...r]);
  const rows = Math.ceil(count / COLS);
  let rem = count;
  for (let i = 0; i < rows; i++) {
    for (let r = 0; r < ROWS - 1; r++) ng[r] = ng[r + 1];
    const fill = Math.min(rem, COLS);
    rem -= fill;
    ng[ROWS - 1] = Array.from({length: COLS}, (_, c) =>
      c < fill ? { color: 'ojama' } : null
    );
  }
  return ng;
}

/** True when any cell in row 0 is occupied — triggers game over. */
const isGameOver = (g) => g[0].some(Boolean);

/* ═══════════════════════════════════════════════════════════════
   §3  MAIN COMPONENT
   ═══════════════════════════════════════════════════════════════ */

export default function PanelBlast() {

  /* ── §3a  React State ─────────────────────────────────────── */
  const [grid,     setGrid    ] = useState(buildGrid);
  const [cursor,   setCursor  ] = useState({ r: 9, c: 2 });
  const [score,    setScore   ] = useState(0);
  const [best,     setBest    ] = useState(0);
  const [phase,    setPhase   ] = useState("start"); // "start"|"play"|"gameover"|"clear"
  // flash state is now encoded in grid cells as { color, flashing:true } — no separate flashSet needed
  const [chainMsg, setChainMsg] = useState(null);    // { text, pts } | null
  const [maxChain,  setMaxChain ] = useState(0);
  const [dragCell,  setDragCell ] = useState(null); // { r, c } | null  ← dragged panel
  const [stage,     setStage    ] = useState(0);
  const [target,    setTarget   ] = useState(TARGET_SCORES[0]);
  const [mode,      setMode     ] = useState("stage"); // "stage"|"timeattack"
  const [timeLeft,  setTimeLeft ] = useState(TIME_ATTACK_SECS); // countdown
  const [bestTA,    setBestTA   ] = useState(0);
  const [riseOffset,  setRiseOffset ] = useState(0);
  const [particles,   setParticles  ] = useState([]);
  const [bgmOn,       setBgmOn      ] = useState(false); // BGM on/off
  const audioCtxRef = useRef(null); // Web Audio API context (ポン音用)
  const bgmNodesRef = useRef([]);   // BGM oscillator nodes (for stop)
  const bgmAudioRef = useRef(null); // <audio> element for MP3 BGM
  const [nextRow,     setNextRow   ] = useState(() => makeNextRow(null));
  const [riseSpeedIdx,setRiseSpeedIdx] = useState(2); // 0=OFF … 5=LV5

  // ── 対戦モード ──
  const [battlePhase,  setBattlePhase ] = useState('off');
  // 'off'|'lobby'|'hosting'|'joining'|'connected'|'error'
  const [myPeerId,     setMyPeerId    ] = useState('');
  const [joinCode,     setJoinCode    ] = useState('');
  const [ojamaQueue,   setOjamaQueue  ] = useState(0);  // 溜まったお邪魔数
  const [opponentInfo, setOpponentInfo] = useState({ score:0, over:false, ojama:0 });

  /* ── §3b  Mutable Refs (avoid stale closures in async logic) ── */
  const gridRef   = useRef(grid);
  const scoreRef  = useRef(0);
  const targetRef = useRef(TARGET_SCORES[0]);
  const modeRef   = useRef('stage'); // mirrors mode
  const cursorRef = useRef(cursor);
  const phaseRef  = useRef(phase);
  const cycleRef    = useRef(null);   // match/clear/chain function (updated each render)
  //
  // ── Active-chain state machine ──────────────────────────────
  //  flashingRef  : true during the flash window → swaps BLOCKED entirely
  //  fallingRef   : true during stagger + CHAIN_DELAY → swaps QUEUED
  //  chainLvlRef  : current chain depth (0 = idle)
  //  (swaps during FALLING are applied immediately to gridRef — tickFall picks them up)
  const flashingRef    = useRef(false);
  const fallingRef     = useRef(false);
  const chainLvlRef    = useRef(0);
  const cycleRunning   = useRef(false);
  const cycleIdRef     = useRef(0);
  const peerRef        = useRef(null);  // PeerJS Peer instance
  const connRef        = useRef(null);  // PeerJS DataConnection
  const ojamaQueueRef  = useRef(0);     // mirrors ojamaQueue for closures
  const battlePhaseRef = useRef('off');
  // pendingSwap removed — falling swaps are now applied immediately

  // Keep refs in sync with latest state values
  useEffect(() => { gridRef.current   = grid;   }, [grid]);
  useEffect(() => { scoreRef.current  = score;  }, [score]);
  useEffect(() => { targetRef.current = target; }, [target]);
  useEffect(() => { modeRef.current      = mode;        }, [mode]);
  useEffect(() => { battlePhaseRef.current = battlePhase; }, [battlePhase]);
  useEffect(() => { ojamaQueueRef.current  = ojamaQueue;  }, [ojamaQueue]);


  /* ── Web Audio API エンジン ──────────────────────────────────────────────────
     外部ライブラリ不要。AudioContext を使って BGM とポン音を生成する。
   ─────────────────────────────────────────────────────────────────────────── */
  const getAudioCtx = useCallback(() => {
    if (!audioCtxRef.current)
      audioCtxRef.current = new (window.AudioContext || window.webkitAudioContext)();
    if (audioCtxRef.current.state === 'suspended')
      audioCtxRef.current.resume();
    return audioCtxRef.current;
  }, []);

  // ── ポン音: i番目のパネルごとにピッチを変える ──
  const playPop = useCallback((i = 0) => {
    try {
      const ctx = getAudioCtx();
      const freqs = [523.25, 587.33, 659.25, 783.99, 880, 1046.5]; // C5〜C6
      const freq = freqs[i % freqs.length];
      const t = ctx.currentTime;
      const master = ctx.createGain();
      master.gain.setValueAtTime(0.35, t);
      master.gain.exponentialRampToValueAtTime(0.001, t + 0.18);
      master.connect(ctx.destination);
      // サイン波 (柔らかいポン)
      const osc1 = ctx.createOscillator();
      osc1.type = 'sine';
      osc1.frequency.setValueAtTime(freq, t);
      osc1.frequency.exponentialRampToValueAtTime(freq * 0.6, t + 0.12);
      osc1.connect(master);
      osc1.start(t); osc1.stop(t + 0.2);
      // 倍音 (少し明るく)
      const osc2 = ctx.createOscillator();
      osc2.type = 'triangle';
      osc2.frequency.setValueAtTime(freq * 2, t);
      osc2.frequency.exponentialRampToValueAtTime(freq, t + 0.08);
      const g2 = ctx.createGain();
      g2.gain.setValueAtTime(0.12, t);
      g2.gain.exponentialRampToValueAtTime(0.001, t + 0.1);
      osc2.connect(g2); g2.connect(ctx.destination);
      osc2.start(t); osc2.stop(t + 0.15);
    } catch(e) {}
  }, [getAudioCtx]);

  // ── BGM: MP3ファイルを <audio> で再生 ──
  const startBgm = useCallback(() => {
    if (!bgmAudioRef.current) {
      const audio = new Audio('/So-Far-So-Good.mp3');
      audio.loop = true;
      audio.volume = 0.55;
      bgmAudioRef.current = audio;
    }
    bgmAudioRef.current.play().catch(() => {});
  }, []);

  const stopBgm = useCallback(() => {
    if (bgmAudioRef.current) {
      bgmAudioRef.current.pause();
      bgmAudioRef.current.currentTime = 0;
    }
  }, []);

  useEffect(() => {
    if (bgmOn) startBgm();
    else stopBgm();
    return () => stopBgm();
  }, [bgmOn]);

  /* ── タイムアタック カウントダウン ── */
  useEffect(() => {
    if (mode !== 'timeattack' || phase !== 'play') return;
    if (timeLeft <= 0) return;
    const id = setTimeout(() => {
      setTimeLeft(t => {
        if (t <= 1) {
          // 時間切れ → timeout フェーズへ
          setBestTA(b => Math.max(b, scoreRef.current));
          setPhase('timeout');
          phaseRef.current = 'timeout';
          return 0;
        }
        return t - 1;
      });
    }, 1000);
    return () => clearTimeout(id);
  }, [mode, phase, timeLeft]);
  useEffect(() => { phaseRef.current  = phase;  }, [phase]);
  useEffect(() => { cursorRef.current = cursor; }, [cursor]);

  const riseOffsetRef = useRef(0);
  const nextRowRef    = useRef(null); // latest nextRow without stale closure
  useEffect(() => { nextRowRef.current = nextRow; }, [nextRow]);

  /* ── 1行コミット: グリッドを1行上にシフトして新行を追加 ── */
  const commitRow = useCallback(() => {
    // フラッシュ・落下・サイクル実行中はグリッドを触らない
    if (flashingRef.current || fallingRef.current || cycleRunning.current) return;
    riseOffsetRef.current = 0;
    setRiseOffset(0);
    const newRow = nextRowRef.current;
    const ng = gridRef.current.map(r => [...r]);
    if (ng[0].some(Boolean)) {
      setPhase('gameover');
      phaseRef.current = 'gameover';
      return;
    }
    for (let r = 0; r < ROWS - 1; r++) ng[r] = ng[r + 1];
    ng[ROWS - 1] = newRow;
    gridRef.current = ng;
    setGrid(ng);
    const next = makeNextRow(newRow);
    nextRowRef.current = next;
    setNextRow(next);
    // サイクルが動いていない時だけ新規起動（二重起動防止）
    if (!cycleRunning.current) {
      setTimeout(() => cycleRef.current(ng, 1), 60);
    }
  }, []);

  /* ── 下から新パネルがゆっくり上昇する ── */
  useEffect(() => {
    if (phase !== 'play') return;
    const id = setInterval(() => {
      // フラッシュ or 落下中は一時停止（操作の邪魔をしない）
      if (flashingRef.current || fallingRef.current) return;

      const px = RISE_SPEEDS[riseSpeedIdx]?.px ?? 0;
      if (px === 0) return; // OFF
      riseOffsetRef.current += px;
      setRiseOffset(riseOffsetRef.current);

      if (riseOffsetRef.current >= CELL) commitRow();
    }, RISE_INTERVAL_MS);
    return () => clearInterval(id);
  }, [phase, riseSpeedIdx]);

  /* ── §3c  Match / Clear / Chain Cycle ─────────────────────────────────────────
     Active-chain state machine:
       IDLE        → chainLvlRef=0, flashingRef=false, fallingRef=false
       FLASHING    → flashingRef=true  (swap BLOCKED)
       FALLING     → fallingRef=true   (swap applied IMMEDIATELY to gridRef.current)
       BETWEEN     → fallingRef=true   (same — swaps go straight into the live grid)
   ─────────────────────────────────────────────────────────────────────────────── */
  cycleRef.current = (g, chain) => {
    chainLvlRef.current = chain;
    cycleRunning.current = true;
    // 各サイクルに一意IDを付与 — 古いタイムアウトが自動キャンセルできる
    cycleIdRef.current  += 1;
    const myCycleId      = cycleIdRef.current;
    // エントリー時点でフラグをクリーンな状態に
    flashingRef.current  = false;
    fallingRef.current   = false;
    // グリッド上に残った flashing:true を全消去（前サイクルの残骸）
    {
      const cleaned = gridRef.current.map(row =>
        row.map(cell => (cell?.flashing ? { color: cell.color } : cell))
      );
      gridRef.current = cleaned;
      setGrid(cleaned);
    }

    // Always read gridRef.current — swaps during CHAIN_DELAY or fall land here,
    // so we never miss a match caused by a mid-animation swap.
    const workG = gridRef.current;

    // お邪魔パネルを隣接消去に含める
    const matches = expandWithOjama(findMatches(workG), workG);

    // ── No matches → chain ends; apply pending ojama if any ──
    if (!matches.size) {
      if (ojamaQueueRef.current > 0) {
        const count = ojamaQueueRef.current;
        ojamaQueueRef.current = 0;
        setOjamaQueue(0);
        const afterOjama = addOjamaToGrid(gridRef.current, count);
        gridRef.current = afterOjama;
        setGrid(afterOjama);
        // お邪魔を追加したのでもう一度マッチチェック
        setTimeout(() => cycleRef.current(afterOjama, 1), CHAIN_DELAY);
        return;
      }
      chainLvlRef.current  = 0;
      fallingRef.current   = false;
      flashingRef.current  = false;
      cycleRunning.current = false;
      return;
    }

    // ── Scoring ──
    //   Base: 10 pts per cleared block × chain multiplier
    //   Chain bonus: (chain-1) × 50 extra points
    const base  = matches.size * 10 * chain;
    const bonus = chain > 1 ? (chain - 1) * 50 : 0;
    const pts   = base + bonus;
    setScore(s => {
      const ns = s + pts;
      setBest(b => Math.max(b, ns));
      scoreRef.current = ns;
      // スコアを対戦相手に送信
      if (connRef.current?.open) connRef.current.send({ type:'score', score:ns });
      // ── クリア判定 ──
      if (ns >= targetRef.current && phaseRef.current === 'play' && modeRef.current === 'stage') {
        setTimeout(() => {
          setPhase('clear');
          phaseRef.current = 'clear';
        }, 400);
      }
      return ns;
    });

    // ── Chain message (shown for chain ≥ 2) ──
    if (chain > 1) {
      setMaxChain(m => Math.max(m, chain));
      const msgKey = Date.now();
      setChainMsg({ text: `✦ Chain ×${chain}!`, pts, key: msgKey });
      setTimeout(() => setChainMsg(m => m?.key === msgKey ? null : m), 1600);
      // 対戦: 連鎖数に応じてお邪魔パネルを送る
      if (battlePhaseRef.current === 'connected' && connRef.current?.open) {
        const ojamaCount = (chain - 1) * 3;
        connRef.current.send({ type: 'ojama', count: ojamaCount });
      }
    }

    // ── Sort matched cells: left→right, top→bottom (wave-pop effect) ──
    const matchArray = [...matches].sort((a, b) => {
      const [ar, ac] = a.split(",").map(Number);
      const [br, bc] = b.split(",").map(Number);
      return ac !== bc ? ac - bc : ar - br;
    });

    // ── CRITICAL: mark ALL matched cells as flashing IN ONE SHOT ──
    // This protects every matched cell from swaps immediately.
    // Visual stagger is handled by CSS animation-delay (flashIndex property),
    // NOT by staggered setTimeout — so no cell can be swapped before being protected.
    flashingRef.current = true;
    fallingRef.current  = false;

    {
      const g = workG.map(row => [...row]);
      matchArray.forEach((k, i) => {
        const [r, c] = k.split(",").map(Number);
        if (g[r][c]) g[r][c] = { ...g[r][c], flashing: true, flashIndex: i };
      });
      gridRef.current = g;
      setGrid(g);
    }

    // ── Delete cells one by one; positions are now stable (all flashing = swap-blocked) ──
    const FALL_STEP_MS = 200;

    matchArray.forEach((k, i) => {
      setTimeout(() => {
        // このタイムアウトが古いサイクルのものであればキャンセル
        if (cycleIdRef.current !== myCycleId) return;

        const [r, c] = k.split(",").map(Number);
        const g = gridRef.current.map(row => [...row]);
        if (g[r][c]?.flashing) {
          // ポン音
          playPop(i);
          // パーティクルを発生させる
          const color = g[r][c].color;
          const cx = c * CELL + CELL / 2;
          const cy = r * CELL + CELL / 2;
          const pid = Date.now() + '_' + r + '_' + c;
          const pts = Array.from({length: 8}, (_, i) => ({
            id: pid + '_' + i,
            x: cx, y: cy,
            angle: (i / 8) * 360,
            color,
          }));
          setParticles(p => [...p.slice(-48), ...pts]);
          setTimeout(() => setParticles(p => p.filter(pt => !pts.some(pp => pp.id === pt.id))), 600);
          g[r][c] = null;
        }
        gridRef.current = g;
        setGrid(g);

        // ── Last cell removed → animate gravity one row at a time ──
        if (i === matchArray.length - 1) {
          flashingRef.current = false;
          fallingRef.current  = true;

          const tickFall = () => {
            // サイクルIDが変わっていたら停止
            if (cycleIdRef.current !== myCycleId) return;

            const { next, moved } = applyGravityStep(gridRef.current);
            setGrid(next);
            gridRef.current = next;
            if (moved) {
              setTimeout(tickFall, FALL_STEP_MS);
            } else {
              if (isGameOver(next)) {
                chainLvlRef.current = 0;
                fallingRef.current  = false;
                cycleRunning.current = false;
                setPhase("gameover");
                phaseRef.current = "gameover";
                return;
              }
              fallingRef.current   = false;
              cycleRunning.current = false;
              setTimeout(() => cycleRef.current(next, chain + 1), CHAIN_DELAY);
            }
          };

          setTimeout(tickFall, FALL_STEP_MS);
        }
      }, i * STEP_MS + CELL_FLASH_MS);
    });
  };


  /* ── §3d  Swap Action ─────────────────────────────────────────────────────────
     Active-chain rules:
       • FLASHING  → ignore (blocks are popping, swap would interfere)
       • FALLING / BETWEEN → queue swap in pendingSwap (applied before next chain check)
       • IDLE      → apply immediately; if chainLvlRef > 0 somehow, continue chain
   ─────────────────────────────────────────────────────────────────────────────── */
  const doSwap = useCallback(() => {
    if (phaseRef.current !== "play") return;
    // 点滅中セルに触れる操作だけをブロック。他の列は自由に操作できる。
    if (flashingRef.current) {
      const { r, c } = cursorRef.current;
      if (c >= COLS - 1) return;
      const g = gridRef.current;
      if (g[r][c]?.flashing || g[r][c + 1]?.flashing) return;
    }

    const { r, c } = cursorRef.current;
    if (c >= COLS - 1) return;

    // ── FALLING or FLASHING (non-flash cell): apply swap immediately ──
    // tickFall reads gridRef.current each tick, so the swap is picked up automatically.
    // We do NOT launch a new cycleRef here — the ongoing cycle will catch any new matches
    // when blocks settle at the end of the current fall sequence.
    if (fallingRef.current || flashingRef.current) {
      const left  = gridRef.current[r][c];
      const right = gridRef.current[r][c + 1];
      if (!left && !right) return;
      const ng = gridRef.current.map(row => [...row]);
      [ng[r][c], ng[r][c + 1]] = [ng[r][c + 1], ng[r][c]];
      setGrid(ng);
      gridRef.current = ng;
      return;
    }

    // ── IDLE: apply immediately and start a new cycle ───────────
    const left  = gridRef.current[r][c];
    const right = gridRef.current[r][c + 1];
    if (!left && !right) return;

    let ng = gridRef.current.map(row => [...row]);
    [ng[r][c], ng[r][c + 1]] = [ng[r][c + 1], ng[r][c]];
    setGrid(ng);
    gridRef.current = ng;

    // スワップ後、ブロックを1行ずつゆっくり落下させる
    fallingRef.current   = true;
    cycleRunning.current = true;
    const FALL_STEP_MS = 200;
    const tickFallIdle = () => {
      const { next, moved } = applyGravityStep(gridRef.current);
      setGrid(next);
      gridRef.current = next;
      if (moved) {
        setTimeout(tickFallIdle, FALL_STEP_MS);
      } else {
        fallingRef.current   = false;
        cycleRunning.current = false;
        setTimeout(() => cycleRef.current(next, 1), 60);
      }
    };
    setTimeout(tickFallIdle, FALL_STEP_MS);
  }, []); // stable — reads only from refs

  /* ── §3e  Keyboard Controls ──────────────────────────────── */
  useEffect(() => {
    const onKey = (e) => {
      if (phaseRef.current !== "play") return;

      const move = {
        ArrowLeft:  () => setCursor(p => { const nc = Math.max(0,        p.c - 1); cursorRef.current = {...p, c: nc}; return {...p, c: nc}; }),
        ArrowRight: () => setCursor(p => { const nc = Math.min(COLS - 2, p.c + 1); cursorRef.current = {...p, c: nc}; return {...p, c: nc}; }),
        ArrowUp:    () => setCursor(p => { const nr = Math.max(0,        p.r - 1); cursorRef.current = {...p, r: nr}; return {...p, r: nr}; }),
        ArrowDown:  () => setCursor(p => { const nr = Math.min(ROWS - 1, p.r + 1); cursorRef.current = {...p, r: nr}; return {...p, r: nr}; }),
      };

      if (move[e.key]) {
        e.preventDefault();
        move[e.key]();
        return;
      }
      if (e.key === " " || e.key === "x" || e.key === "X") {
        e.preventDefault();
        doSwap();
      }
    };

    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [doSwap]);

  /* ── §3f  Game Start / Reset ─────────────────────────────── */
  const startGame = useCallback((stageIdx = 0) => {
    const ng = buildGrid();
    setGrid(ng);
    gridRef.current = ng;

    const initialCursor = { r: 9, c: 2 };
    setCursor(initialCursor);
    cursorRef.current = initialCursor;

    const t = TARGET_SCORES[stageIdx] ?? TARGET_SCORES[TARGET_SCORES.length - 1];
    setStage(stageIdx);
    setTarget(t);
    targetRef.current = t;
    setScore(0);
    scoreRef.current = 0;
    setMaxChain(0);
    setChainMsg(null);
    flashingRef.current = false;
    fallingRef.current  = false;
    chainLvlRef.current = 0;
    riseOffsetRef.current = 0;
    setRiseOffset(0);
    const firstNext = makeNextRow(null);
    nextRowRef.current = firstNext;
    setNextRow(firstNext);
    setMode('stage');
    modeRef.current = 'stage';
    setPhase("play");
    phaseRef.current = "play";
  }, []);

  /* ── タイムアタック開始 ── */
  const startTimeAttack = useCallback(() => {
    const ng = buildGrid();
    setGrid(ng); gridRef.current = ng;
    const ic = { r: 9, c: 2 };
    setCursor(ic); cursorRef.current = ic;
    setScore(0); scoreRef.current = 0;
    setMaxChain(0);
    setChainMsg(null);
    setTimeLeft(TIME_ATTACK_SECS);
    flashingRef.current = false;
    fallingRef.current  = false;
    chainLvlRef.current = 0;
    riseOffsetRef.current = 0;
    setRiseOffset(0);
    const firstNextTA = makeNextRow(null);
    nextRowRef.current = firstNextTA;
    setNextRow(firstNextTA);
    setMode('timeattack');
    modeRef.current = 'timeattack';
    setPhase('play');
    phaseRef.current = 'play';
  }, []);

  /* ── §3h  対戦 PeerJS ──────────────────────────────────────────────────────
     PeerJS (WebRTC P2P) を使ってサーバーレスで2人対戦。
     Player1 がホスト → Peer ID を相手に共有
     Player2 がその ID で接続 → 双方向 DataChannel 確立
   ─────────────────────────────────────────────────────────────────────────── */
  const setupConn = (conn) => {
    conn.on('open', () => {
      setBattlePhase('connected');
      battlePhaseRef.current = 'connected';
    });
    conn.on('data', (data) => {
      if (data.type === 'ojama') {
        ojamaQueueRef.current += data.count;
        setOjamaQueue(q => q + data.count);
        setOpponentInfo(p => ({ ...p, ojama: (p.ojama||0) + data.count }));
      }
      if (data.type === 'score') {
        setOpponentInfo(p => ({ ...p, score: data.score }));
      }
      if (data.type === 'gameover') {
        setOpponentInfo(p => ({ ...p, over: true }));
      }
      if (data.type === 'start') {
        // ホストから「試合開始」の合図
        startBattleGame();
      }
    });
    conn.on('close', () => {
      setBattlePhase('error');
      battlePhaseRef.current = 'error';
    });
  };

  const destroyPeer = () => {
    connRef.current?.close();
    peerRef.current?.destroy();
    connRef.current = null;
    peerRef.current = null;
  };

  const startBattleGame = useCallback(() => {
    ojamaQueueRef.current = 0;
    setOjamaQueue(0);
    setOpponentInfo({ score:0, over:false, ojama:0 });
    startTimeAttack();
  }, [startTimeAttack]);

  const generateRoomCode = () => {
    // 読み間違えやすい文字(0/O, 1/I/L)を除いた英数字で6文字のコードを生成
    const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
    return Array.from({ length: 6 }, () => chars[Math.floor(Math.random() * chars.length)]).join('');
  };

  const hostBattle = useCallback(() => {
    setBattlePhase('hosting');
    setMyPeerId('');
    try {
      // PeerJS を CDN から動的ロード
      // PeerJS は index.html の <script> タグで読み込み済み
      const Peer = window.Peer;
      if (!Peer) { setBattlePhase('error'); return; }
      const shortCode = generateRoomCode();
      const peer = new Peer(shortCode, {
        host: '0.peerjs.com',
        port: 443,
        path: '/',
        secure: true,
        debug: 0,
        config: { iceServers: [
          { urls: 'stun:stun.l.google.com:19302' },
          { urls: 'stun:stun1.l.google.com:19302' },
          { urls: 'stun:stun.relay.metered.ca:80' },
          { urls: 'turn:global.relay.metered.ca:80', username: 'openrelayproject', credential: 'openrelayproject' },
          { urls: 'turn:global.relay.metered.ca:443', username: 'openrelayproject', credential: 'openrelayproject' },
          { urls: 'turn:global.relay.metered.ca:80?transport=tcp', username: 'openrelayproject', credential: 'openrelayproject' },
        ]},
      });
      peerRef.current = peer;
      // 15秒でタイムアウト
      const timeout = setTimeout(() => {
        if (!myPeerId) setBattlePhase('error');
      }, 15000);
      peer.on('open', id => {
        clearTimeout(timeout);
        setMyPeerId(id);
      });
      peer.on('connection', conn => {
        connRef.current = conn;
        setupConn(conn);
        setTimeout(() => {
          conn.send({ type: 'start' });
          startBattleGame();
        }, 500);
      });
      peer.on('error', (err) => {
        console.error('PeerJS error:', err);
        setBattlePhase('error');
      });
    } catch(e) {
      console.error('PeerJS load error:', e);
      setBattlePhase('error');
    }
  }, [startBattleGame, myPeerId]);

  const joinBattle = useCallback((code) => {
    if (!code.trim()) return;
    setBattlePhase('joining');
    try {
      const Peer = window.Peer;
      if (!Peer) { setBattlePhase('error'); return; }
      const peer = new Peer(undefined, {
        host: '0.peerjs.com', port: 443, path: '/', secure: true, debug: 0,
        config: { iceServers: [
          { urls: 'stun:stun.l.google.com:19302' },
          { urls: 'stun:stun1.l.google.com:19302' },
          { urls: 'stun:stun.relay.metered.ca:80' },
          { urls: 'turn:global.relay.metered.ca:80', username: 'openrelayproject', credential: 'openrelayproject' },
          { urls: 'turn:global.relay.metered.ca:443', username: 'openrelayproject', credential: 'openrelayproject' },
          { urls: 'turn:global.relay.metered.ca:80?transport=tcp', username: 'openrelayproject', credential: 'openrelayproject' },
        ]},
      });
      peerRef.current = peer;
      const timeout = setTimeout(() => setBattlePhase('error'), 15000);
      peer.on('open', () => {
        clearTimeout(timeout);
        const conn = peer.connect(code.trim().toUpperCase());
        connRef.current = conn;
        setupConn(conn);
        const connTimeout = setTimeout(() => {
          if (!conn.open) setBattlePhase('error');
        }, 20000);
        conn.on('open', () => clearTimeout(connTimeout));
      });
      peer.on('error', (err) => {
        console.error('PeerJS error:', err);
        setBattlePhase('error');
      });
    } catch(e) {
      setBattlePhase('error');
    }
  }, []);

  const leaveBattle = useCallback(() => {
    destroyPeer();
    setBattlePhase('off');
    battlePhaseRef.current = 'off';
    setMyPeerId('');
    setJoinCode('');
    setPhase('start');
    phaseRef.current = 'start';
  }, []);

  // ゲームオーバー時に対戦相手へ通知
  useEffect(() => {
    if (phase === 'gameover' && battlePhaseRef.current === 'connected') {
      connRef.current?.send({ type: 'gameover' });
    }
  }, [phase]);

  /* ── §3g  Pointer / Touch Controls ────────────────────────────────────────────
     Drag system:
       pointerdown  → "grab" the panel, record column + threshold position
       pointermove  → every time finger crosses ±(CELL/2)px from last threshold,
                      set cursor to the correct column and call doSwap()
                      then advance grabCol and reset threshold
       pointerup    → release grab
     This lets players slide a panel left or right across multiple cells
     in one continuous drag gesture.
   ─────────────────────────────────────────────────────────────────────────── */
  const dragRef = useRef(null);
  // dragRef: { r, grabCol, threshX }
  //   grabCol  = current column of the grabbed panel
  //   threshX  = clientX of the last swap trigger (advances by CELL/2 each swap)

  const onPointerDown = (e, r, c) => {
    if (phaseRef.current !== "play") return;
    e.currentTarget.setPointerCapture(e.pointerId);
    // Move cursor to this row/col (clamped so right partner is always in-bounds)
    const nc = Math.min(c, COLS - 2);
    setCursor({ r, c: nc });
    cursorRef.current = { r, c: nc };
    dragRef.current = { r, grabCol: c, threshX: e.clientX };
    setDragCell({ r, c });
  };

  const onPointerMove = (e) => {
    const d = dragRef.current;
    if (!d) return;

    const dx = e.clientX - d.threshX;

    if (dx >= CELL * 1.2 && d.grabCol < COLS - 1) {
      // ── Dragged right: cursor LEFT = grabCol, swap rightward ──
      const nc = Math.min(d.grabCol, COLS - 2);
      cursorRef.current = { r: d.r, c: nc };
      setCursor({ r: d.r, c: nc });
      doSwap();
      d.grabCol   += 1;
      d.threshX   += CELL * 1.2;
      setDragCell({ r: d.r, c: d.grabCol });

    } else if (dx <= -(CELL * 1.2) && d.grabCol > 0) {
      // ── Dragged left: cursor LEFT = grabCol-1, swap leftward ──
      const nc = Math.min(d.grabCol - 1, COLS - 2);
      cursorRef.current = { r: d.r, c: nc };
      setCursor({ r: d.r, c: nc });
      doSwap();
      d.grabCol   -= 1;
      d.threshX   -= CELL * 1.2;
      setDragCell({ r: d.r, c: d.grabCol });
    }
  };

  const onPointerUp = () => {
    dragRef.current = null;
    setDragCell(null);
  };

  /* ═══════════════════════════════════════════════════════════
     §4  RENDER
   ═══════════════════════════════════════════════════════════ */

  const boardW = COLS * CELL;
  const boardH = ROWS * CELL;

  return (
    <div style={S.page}>
      {/* CSS keyframes injected once */}
      <style>{KEYFRAMES}</style>

      {/* ── HEADER ── */}
      <header style={S.header}>
        {/* タイトルはプレイ中は非表示 */}
        {phase !== 'play' && (
          <div style={S.titleBlock}>
            <span style={S.titleTop}>PANEL</span>
            <span style={S.titleBot}>BLAST</span>
          </div>
        )}

        {/* スコア + タイマー / プログレス */}
        <div style={{display:"flex", flexDirection:"column", gap:6, alignItems:"center"}}>

          {/* タイムアタック: 残り時間を大きく表示 */}
          {mode === 'timeattack' && phase === 'play' && (
            <div style={S.timerBig}>
              <span style={{
                ...S.timerDigits,
                color: timeLeft <= 10 ? '#FF3B30' : timeLeft <= 20 ? '#FF9500' : '#fff',
                textShadow: timeLeft <= 10
                  ? '0 0 24px #FF3B30'
                  : timeLeft <= 20
                    ? '0 0 16px #FF9500'
                    : '0 0 12px rgba(255,255,255,0.4)',
              }}>
                {String(Math.floor(timeLeft/60)).padStart(1,'0')}:{String(timeLeft%60).padStart(2,'0')}
              </span>
              {/* タイマーバー */}
              <div style={{...S.progressWrap, width:'100%', marginTop:2}}>
                <div style={{...S.progressBar,
                  width: `${timeLeft/TIME_ATTACK_SECS*100}%`,
                  background: timeLeft <= 10
                    ? 'linear-gradient(90deg,#FF3B30,#FF9500)'
                    : 'linear-gradient(90deg,#5AC8FA,#30D158)',
                  boxShadow: timeLeft <= 10 ? '0 0 8px #FF3B3088' : '0 0 8px #5AC8FA88',
                  transition: 'width 0.9s linear, background 0.3s',
                }} />
              </div>
            </div>
          )}

          <div style={S.scorePair}>
            {mode === 'timeattack'
              ? <ScoreCard icon={"⚡"} label="SCORE" value={score} color="#FF9500"/>
              : <ScoreCard icon={"⚡"} label={`STAGE ${stage+1} SCORE`} value={score} color="#FFD60A"/>
            }
            {mode === 'timeattack'
              ? <ScoreCard icon={"🏆"} label="BEST TA" value={bestTA} color="#FF9500"/>
              : <ScoreCard icon={"🏆"} label="BEST" value={best} color="#5AC8FA"/>
            }
          </div>

          {/* ステージモードのプログレスバー */}
          {mode !== 'timeattack' && (
            <div style={S.progressWrap}>
              <div style={{...S.progressBar, width: `${Math.min(score/target*100,100)}%`}} />
              <span style={S.progressLabel}>TARGET {String(target).padStart(7,"0")}</span>
            </div>
          )}
        </div>

        {/* Max chain badge */}
        {maxChain > 1 && (
          <div style={S.chainBadge}>
            MAX ×{maxChain}
          </div>
        )}
      </header>

      {/* ── GAME BOARD + PUSH ラッパー ── */}
      <div style={{ position: "relative", display: "inline-block" }}>

      {/* ── タイムアタック: 左側縦タイマー ── */}
      {mode === "timeattack" && phase === "play" && (
        <div style={{
          position: "absolute",
          left: -44, top: 0, bottom: 0,
          width: 36,
          display: "flex",
          flexDirection: "column",
          alignItems: "center",
          justifyContent: "space-between",
          paddingTop: 8, paddingBottom: 8,
          gap: 4,
          zIndex: 8,
        }}>
          {/* 残り時間 数字 */}
          <div style={{
            display: "flex",
            flexDirection: "column",
            alignItems: "center",
            gap: 2,
          }}>
            <span style={{
              fontSize: 9,
              letterSpacing: 1,
              color: "#4A4A88",
              fontFamily: "'Courier New', monospace",
              writingMode: "vertical-rl",
              textOrientation: "mixed",
            }}>TIME</span>
            <span style={{
              fontSize: 22,
              fontWeight: 900,
              fontFamily: "'Courier New', monospace",
              color: timeLeft <= 10 ? "#FF3B30"
                   : timeLeft <= 20 ? "#FF9500" : "#5AC8FA",
              textShadow: timeLeft <= 10 ? "0 0 12px #FF3B30"
                        : timeLeft <= 20 ? "0 0 12px #FF9500" : "0 0 12px #5AC8FA",
              lineHeight: 1,
              animation: timeLeft <= 10 ? "pulse 0.5s infinite" : "none",
            }}>{String(timeLeft).padStart(2,"0")}</span>
          </div>
          {/* 縦のバー */}
          <div style={{
            flex: 1,
            width: 8,
            background: "#0D0D25",
            border: "1px solid #1E1E4A",
            borderRadius: 99,
            position: "relative",
            overflow: "hidden",
            margin: "6px 0",
          }}>
            <div style={{
              position: "absolute",
              bottom: 0, left: 0, right: 0,
              height: `${timeLeft / TIME_ATTACK_SECS * 100}%`,
              background: timeLeft <= 10
                ? "linear-gradient(0deg,#FF3B30,#FF9500)"
                : "linear-gradient(0deg,#5AC8FA,#30D158)",
              borderRadius: 99,
              transition: "height 0.9s linear, background 0.3s",
              boxShadow: timeLeft <= 10 ? "0 0 8px #FF3B3099" : "0 0 8px #5AC8FA99",
            }}/>
          </div>
          <span style={{
            fontSize: 9, color: "#4A4A88",
            fontFamily: "'Courier New', monospace",
            writingMode: "vertical-rl",
          }}>LEFT</span>
        </div>
      )}

      {/* ── PUSH UP ボタン: ボード右下外側 ── */}
      {phase === "play" && (
        <button
          style={{
            position: "absolute",
            right: -36, bottom: 8,
            zIndex: 8,
            background: "linear-gradient(135deg,#BF5AF2,#581C87)",
            border: "1px solid #DDA0F966",
            borderRadius: 6,
            color: "#fff",
            fontSize: 7,
            fontWeight: 700,
            letterSpacing: 0,
            padding: "10px 4px",
            cursor: "pointer",
            fontFamily: "'Courier New', monospace",
            boxShadow: "0 0 10px #BF5AF266",
            userSelect: "none",
            touchAction: "none",
            display: "flex",
            flexDirection: "column",
            alignItems: "center",
            lineHeight: 1.1,
            gap: 0,
            minWidth: 28,
          }}
          onPointerDown={(e) => {
            e.currentTarget.setPointerCapture(e.pointerId);
            if (phaseRef.current !== "play") return;
            riseOffsetRef.current = CELL;
            commitRow();
            const iv = setInterval(() => {
              if (phaseRef.current !== "play") { clearInterval(iv); return; }
              riseOffsetRef.current = CELL;
              commitRow();
            }, 300);
            window.__pushIv = iv;
          }}
          onPointerUp={()   => clearInterval(window.__pushIv)}
          onPointerLeave={() => clearInterval(window.__pushIv)}
          title="パネルを1行上げる (長押しで連続)"
        >
          ↑
UP
        </button>
      )}

      {/* ── GAME BOARD ── */}
      <div
        style={{ ...S.board, width: boardW, height: boardH }}
        onPointerMove={onPointerMove}
        onPointerUp={onPointerUp}
        onPointerLeave={onPointerUp}
      >
        {/* ── Rising wrapper: translateY pushes content up smoothly ── */}
        <div style={{ position:'absolute', inset:0,
          transform: `translateY(${-riseOffset}px)`,
          willChange: 'transform' }}>

        {/* ── Next row preview (sits just below the board, rises into view) ── */}
        {nextRow.map((cell, c) => {
          const pal = cell ? PALETTE[cell.color] : null;
          return (
            <div key={`next-${c}`} style={{
              position:'absolute',
              left: c * CELL + 2, top: ROWS * CELL + 2,
              width: CELL - 4, height: CELL - 4,
              borderRadius: 8, opacity: 0.5,
              background: pal ? `linear-gradient(145deg,${pal.hi},${pal.mid},${pal.lo})` : 'transparent',
              border: pal ? `2px solid ${pal.hi}60` : 'none',
              display:'flex', alignItems:'center', justifyContent:'center',
              fontSize: 16, color:'#ffffff88',
            }}>{pal?.icon}</div>
          );
        })}

        {/* ── Grid cells ── */}
        {grid.map((row, r) =>
          row.map((cell, c) => {
            const key     = `${r},${c}`;
            const pal     = cell ? PALETTE[cell.color] : null;
            const flashing = !!cell?.flashing; // encoded in grid cell — atomic with setGrid

            return (
              <div
                key={key}
                className={flashing ? "cell-flash" : ""}
                style={(() => {
                  const grabbed = dragCell && dragCell.r === r && dragCell.c === c;
                  return {
                    position:  "absolute",
                    left:       c * CELL + 2,
                    top:        r * CELL + 2,
                    width:      CELL - 4,
                    height:     CELL - 4,
                    borderRadius: 8,
                    background: pal
                      ? `linear-gradient(145deg, ${pal.hi} 0%, ${pal.mid} 50%, ${pal.lo} 100%)`
                      : "rgba(255,255,255,0.025)",
                    border: grabbed
                      ? `3px solid #FFFFFF`
                      : pal
                        ? `2px solid ${pal.hi}90`
                        : "1px solid rgba(255,255,255,0.06)",
                    boxShadow: grabbed
                      ? `0 0 0 3px rgba(255,255,255,0.4), 0 4px 18px ${pal?.mid ?? "#fff"}99`
                      : pal
                        ? `0 2px 10px ${pal.mid}55, inset 0 1px 0 ${pal.hi}66`
                        : "none",
                    transform: grabbed ? "scale(1.18)" : "scale(1)",
                    zIndex:    grabbed ? 6 : 1,
                    display: "flex",
                    alignItems: "center",
                    justifyContent: "center",
                    fontSize: 18,
                    color: "#fff",
                    textShadow: pal ? `0 1px 4px ${pal.lo}, 0 0 8px ${pal.mid}88` : "none",
                    cursor: "grab",
                    userSelect: "none",
                    touchAction: "none",
                    willChange: "transform, opacity",
                    transition: grabbed ? "none" : "transform 0.08s, box-shadow 0.08s",
                    // Stagger the flash animation start per cell index
                    animationDelay: (cell?.flashIndex ?? 0) * STEP_MS + "ms",
                  };
                })()}
                onPointerDown={(e) => onPointerDown(e, r, c)}
              >
                {pal?.icon}
              </div>
            );
          })
        )}

        </div>{/* end rising wrapper */}

        {/* ── Cursor bracket (2-cell wide, animated glow) ── */}
        {phase === "play" && (
          <div
            className="cursor-glow"
            style={{
              position:      "absolute",
              left:           cursor.c * CELL + 1,
              top:            cursor.r * CELL + 1 - riseOffset,
              width:          CELL * 2 - 2,
              height:         CELL - 2,
              border:         "3px solid #FFFFFF",
              borderRadius:   10,
              pointerEvents:  "none",
              zIndex:         5,
            }}
          />
        )}

        {/* ── Burst particles ── */}
        {particles.map(pt => {
          const pal = PALETTE[pt.color];
          return (
            <div key={pt.id} className="burst-particle" style={{
              position: 'absolute',
              left: pt.x, top: pt.y,
              width: 7, height: 7,
              borderRadius: '50%',
              background: pal?.mid ?? '#fff',
              boxShadow: `0 0 6px ${pal?.glow ?? '#fff'}`,
              pointerEvents: 'none',
              zIndex: 9,
              '--angle': `${pt.angle}deg`,
            }} />
          );
        })}

        {/* ── Chain popup ── */}
        {chainMsg && (
          <div key={chainMsg.key} className="chain-popup" style={S.chainWrap}>
            <div style={S.chainText}>{chainMsg.text}</div>
            <div style={S.chainPts}>+{chainMsg.pts} pts</div>
          </div>
        )}

        {/* ── Start / Game-over overlay ── */}
        {phase !== "play" && (
          <div style={S.overlay}>
            <div style={S.overlayCard}>
              {/* タイトル */}
              <div style={{...S.overlayTitle,
                color: phase==="clear" ? "#FFD60A" : phase==="timeout" ? "#FF9500" : "#fff",
                textShadow: phase==="clear" ? "0 0 30px #FFD60A" : phase==="timeout" ? "0 0 30px #FF9500" : "0 0 30px #5AC8FA"}}>
                { phase==="gameover" ? "GAME OVER"
                : phase==="clear"    ? "★ STAGE CLEAR!"
                : phase==="timeout"  ? "⏱ TIME UP!"
                :                       "PANEL BLAST" }
              </div>

              {/* スコア表示 */}
              {(phase==="gameover"||phase==="clear"||phase==="timeout") && (
                <>
                  <div style={S.overlayScore}>{String(score).padStart(7,"0")}</div>
                  {phase==="timeout" && bestTA > 0 && score >= bestTA && (
                    <div style={S.overlayNewBest}>🏆 New Best!</div>
                  )}
                  {phase!=="timeout" && best > 0 && score >= best && (
                    <div style={S.overlayNewBest}>★ New Best Score!</div>
                  )}
                </>
              )}

              {/* ヒント / 説明 */}
              <div style={S.overlayHint}>
                { phase==="gameover" ? "Blocks reached the top!"
                : phase==="clear"    ? (stage+1<TARGET_SCORES.length ? `Stage ${stage+1} complete! Next: ${TARGET_SCORES[stage+1].toLocaleString()} pts` : "🎉 All stages cleared!")
                : phase==="timeout"  ? `1分間スコアアタック終了！ベスト: ${bestTA.toLocaleString()} pts`
                : /* start */ "" }
              </div>

              {/* スタート画面: モード選択 */}
              {phase==="start" && (
                <div style={{display:"flex",flexDirection:"column",gap:10,marginTop:8}}>
                  <div style={{fontSize:11,color:"#444477",letterSpacing:2,marginBottom:4}}>MODE SELECT</div>
                  <button style={{...S.overlayBtn}} onClick={()=>startGame(0)}
                    onMouseEnter={e=>e.currentTarget.style.filter="brightness(1.2)"}
                    onMouseLeave={e=>e.currentTarget.style.filter="brightness(1)"}>
                    🎯 STAGE MODE
                  </button>
                  <button style={{...S.overlayBtn, background:"linear-gradient(135deg,#FF9500,#FF3B30)"}}
                    onClick={startTimeAttack}
                    onMouseEnter={e=>e.currentTarget.style.filter="brightness(1.2)"}
                    onMouseLeave={e=>e.currentTarget.style.filter="brightness(1)"}>
                    ⏱ 1MIN SCORE ATTACK
                  </button>
                  <button style={{...S.overlayBtn, background:"linear-gradient(135deg,#FF2D55,#8B001A)"}}
                    onClick={()=>setBattlePhase("lobby")}
                    onMouseEnter={e=>e.currentTarget.style.filter="brightness(1.2)"}
                    onMouseLeave={e=>e.currentTarget.style.filter="brightness(1)"}>
                    ⚔ BATTLE (P2P)
                  </button>
                </div>
              )}

              {/* NEXT STAGE */}
              {phase==="clear" && stage+1<TARGET_SCORES.length && (
                <button style={{...S.overlayBtn,background:"linear-gradient(135deg,#FFD60A,#FF9500)",color:"#07071A",marginBottom:8}}
                  onClick={()=>startGame(stage+1)}
                  onMouseEnter={e=>e.currentTarget.style.filter="brightness(1.15)"}
                  onMouseLeave={e=>e.currentTarget.style.filter="brightness(1)"}>
                  ▶ NEXT STAGE
                </button>
              )}

              {/* もう一度 / 戻る ボタン */}
              {phase!=="start" && (
                <div style={{display:"flex",flexDirection:"column",gap:8,marginTop:4}}>
                  {(phase==="gameover"||phase==="timeout") && (
                    <button style={{...S.overlayBtn,background:phase==="timeout"?"linear-gradient(135deg,#FF9500,#FF3B30)":undefined}}
                      onClick={phase==="timeout" ? startTimeAttack : ()=>startGame(0)}
                      onMouseEnter={e=>e.currentTarget.style.filter="brightness(1.2)"}
                      onMouseLeave={e=>e.currentTarget.style.filter="brightness(1)"}>
                      ▶ {phase==="timeout" ? "PLAY AGAIN" : "PLAY AGAIN"}
                    </button>
                  )}
                  <button style={{...S.overlayBtn, background:"linear-gradient(135deg,#3A3A6A,#1A1A3A)", fontSize:12}}
                    onClick={()=>{setPhase('start');phaseRef.current='start';}}
                    onMouseEnter={e=>e.currentTarget.style.filter="brightness(1.3)"}
                    onMouseLeave={e=>e.currentTarget.style.filter="brightness(1)"}>
                    ↩ MODE SELECT
                  </button>
                </div>
              )}
            </div>
          </div>
        )}
      </div>
      </div>{/* end board+push wrapper */}

      {/* ── 対戦ロビー オーバーレイ ── */}
      {battlePhase !== "off" && phase === "start" && (
        <div style={{...S.overlay, position:"fixed", zIndex:100}}>
          <div style={{...S.overlayCard, maxWidth:300}}>
            <div style={{...S.overlayTitle, fontSize:22, marginBottom:12}}>⚔ BATTLE</div>

            {battlePhase === "lobby" && (
              <>
                <div style={S.overlayHint}>相手と接続してリアルタイム対戦！<br/>連鎖でお邪魔パネルを送り合う。</div>
                <div style={{display:"flex",flexDirection:"column",gap:8,marginTop:8}}>
                  <button style={{...S.overlayBtn,background:"linear-gradient(135deg,#30D158,#14532D)"}}
                    onClick={hostBattle}
                    onMouseEnter={e=>e.currentTarget.style.filter="brightness(1.2)"}
                    onMouseLeave={e=>e.currentTarget.style.filter="brightness(1)"}>
                    🏠 部屋を作る（ホスト）
                  </button>
                  <div style={{display:"flex",gap:6,alignItems:"center"}}>
                    <input
                      placeholder="ルームコードを入力"
                      value={joinCode}
                      onChange={e=>setJoinCode(e.target.value.toUpperCase())}
                      style={{flex:1,background:"#0D0D25",border:"1px solid #1E1E4A",
                        borderRadius:6,color:"#fff",padding:"8px 10px",
                        fontFamily:"inherit",fontSize:11,outline:"none"}}
                    />
                    <button style={{...S.overlayBtn,padding:"8px 12px",fontSize:12}}
                      onClick={()=>joinBattle(joinCode)}
                      onMouseEnter={e=>e.currentTarget.style.filter="brightness(1.2)"}
                      onMouseLeave={e=>e.currentTarget.style.filter="brightness(1)"}>
                      参加
                    </button>
                  </div>
                </div>
              </>
            )}

            {battlePhase === "hosting" && (
              <>
                <div style={S.overlayHint}>このコードを相手に送ってください</div>
                <div style={{background:"#0D0D25",border:"1px solid #30D158",borderRadius:8,
                  padding:"10px 14px",fontSize:11,letterSpacing:1,color:"#30D158",
                  fontFamily:"monospace",wordBreak:"break-all",marginBottom:8,
                  minHeight:40,display:"flex",alignItems:"center",justifyContent:"center"}}>
                  {myPeerId
                    ? myPeerId
                    : <span style={{color:"#4A4A88",fontSize:11,animation:"pulse 1s infinite"}}>
                        ⏳ シグナリングサーバーに接続中...
                      </span>
                  }
                </div>
                {myPeerId && (
                  <button style={{...S.overlayBtn,fontSize:11,marginBottom:8}}
                    onClick={()=>navigator.clipboard?.writeText(myPeerId)}>
                    📋 コピー
                  </button>
                )}
                <div style={{fontSize:11,color:"#4A4A88"}}>相手の接続を待っています...</div>
              </>
            )}

            {battlePhase === "joining" && (
              <div style={{fontSize:12,color:"#5AC8FA"}}>接続中...</div>
            )}

            {battlePhase === "error" && (
              <>
                <div style={{fontSize:12,color:"#FF3B30",marginBottom:8,lineHeight:1.6}}>
                  接続できませんでした。<br/>
                  ネットワーク環境をご確認ください。<br/>
                  <span style={{fontSize:10,color:"#666699"}}>
                    (PeerJSサーバーへの接続が必要です)
                  </span>
                </div>
                <button style={S.overlayBtn} onClick={()=>setBattlePhase("lobby")}>再試行</button>
              </>
            )}

            <button style={{...S.overlayBtn,background:"#1A1A3A",marginTop:12,fontSize:11}}
              onClick={leaveBattle}
              onMouseEnter={e=>e.currentTarget.style.filter="brightness(1.3)"}
              onMouseLeave={e=>e.currentTarget.style.filter="brightness(1)"}>
              ↩ 戻る
            </button>
          </div>
        </div>
      )}

      {/* ── 対戦中 相手HUD ── */}
      {battlePhase === "connected" && phase === "play" && (
        <div style={S.opponentHud}>
          <div style={S.opponentTitle}>⚔ 相手</div>
          <div style={{color:"#FF9500",fontSize:16,fontWeight:700,letterSpacing:1,fontFamily:"monospace"}}>
            {String(opponentInfo.score).padStart(7,"0")}
          </div>
          {opponentInfo.ojama > 0 && (
            <div style={{fontSize:10,color:"#FF3B30",marginTop:2}}>💀 ojama×{opponentInfo.ojama}</div>
          )}
          {opponentInfo.over && (
            <div style={{fontSize:11,color:"#30D158",marginTop:4}}>✓ OVER</div>
          )}
          {ojamaQueue > 0 && (
            <div style={{fontSize:10,color:"#FF2D55",marginTop:4}}>受信 ✕{ojamaQueue}</div>
          )}
          <button style={{...S.btn,fontSize:9,padding:"4px 6px",marginTop:8,
            background:"#1A1A3A"}} onClick={leaveBattle}>切断</button>
        </div>
      )}

      {/* ── CONTROL BAR ── */}
      <div style={S.controlBar}>
        {/* Key hints */}
        <div style={S.hints}>
          {[
            ["↑↓←→", "Move"],
            ["SPC / X", "Swap"],
          ].map(([key, label]) => (
            <span key={key} style={S.hint}>
              <span style={S.kbd}>{key}</span> {label}
            </span>
          ))}
        </div>

        {/* 上昇速度セレクター */}
        <div style={S.speedRow}>
          <span style={S.speedLabel}>⬆ SPEED</span>
          {RISE_SPEEDS.map((s, i) => (
            <button key={i}
              style={{
                ...S.speedBtn,
                background: i === riseSpeedIdx
                  ? 'linear-gradient(135deg,#5AC8FA,#007AFF)'
                  : '#0D0D25',
                border: i === riseSpeedIdx
                  ? '1px solid #5AC8FA'
                  : '1px solid #1E1E4A',
                color: i === riseSpeedIdx ? '#fff' : '#4A4A88',
                boxShadow: i === riseSpeedIdx ? '0 0 8px #5AC8FA66' : 'none',
              }}
              onClick={() => setRiseSpeedIdx(i)}
            >{s.label}</button>
          ))}
        </div>

        {/* Action buttons */}
        <div style={S.btnRow}>
          <button
            style={{ ...S.btn, background: bgmOn
              ? "linear-gradient(135deg,#FFD60A,#78450B)"
              : "linear-gradient(135deg,#3A3A6A,#1A1A3A)",
              boxShadow: bgmOn ? "0 0 10px #FFD60A66" : "none" }}
            onClick={() => setBgmOn(v => !v)}
            title="BGM on/off"
          >
            {bgmOn ? "🎵 BGM" : "🔇 BGM"}
          </button>
          <button
            style={{ ...S.btn, background: "linear-gradient(135deg,#30D158,#14532D)" }}
            onClick={doSwap}
            title="Swap (Space)"
          >
            "⇄" SWAP
          </button>

          <button
            style={{ ...S.btn, background: "linear-gradient(135deg,#FF3B30,#7F1D1D)" }}
            onClick={() => mode === "timeattack" ? startTimeAttack() : startGame(stage)}
            title="Reset game"
          >
            "↺" RESET
          </button>
        </div>
      </div>
    </div>
  );
}

/* ── Small sub-component: Score card ── */
function ScoreCard({ icon, label, value, color }) {
  return (
    <div style={SC.card}>
      <div style={SC.lbl}>{icon} {label}</div>
      <div style={{ ...SC.val, color }}>{String(value).padStart(7, "0")}</div>
    </div>
  );
}
const SC = {
  card: { background:"#0D0D25", border:"1px solid #1E1E4A", borderRadius:6, padding:"4px 10px", textAlign:"center", minWidth:110 },
  lbl:  { fontSize:9, letterSpacing:2, color:"#4A4A88", display:"flex", alignItems:"center", gap:3, justifyContent:"center", marginBottom:2 },
  val:  { fontSize:16, fontWeight:700, letterSpacing:2, fontFamily:"'Courier New', monospace" },
};

/* ═══════════════════════════════════════════════════════════════
   §5  STYLES
   ═══════════════════════════════════════════════════════════════ */

const S = {
  /* layout */
  page: {
    minHeight: "100vh",
    background: "#07071A",
    display: "flex",
    flexDirection: "column",
    alignItems: "center",
    justifyContent: "center",
    padding: 16,
    fontFamily: "'Courier New', Courier, monospace",
    color: "#fff",
  },

  /* header */
  header: {
    display: "flex",
    alignItems: "center",
    gap: 16,
    marginBottom: 10,
    flexWrap: "wrap",
    justifyContent: "center",
  },
  titleBlock: { display:"flex", flexDirection:"column", lineHeight:1 },
  titleTop:   { fontSize:30, fontWeight:900, letterSpacing:6, color:"#fff", textShadow:"0 0 22px #5AC8FA" },
  titleBot:   { fontSize:16, fontWeight:900, letterSpacing:10, color:"#FFD60A", textShadow:"0 0 14px #FFD60A88" },
  scorePair:  { display:"flex", gap:8 },
  progressWrap: {
    position: "relative", width: "100%", maxWidth: 230,
    height: 10, background: "#0D0D25",
    border: "1px solid #1E1E4A", borderRadius: 99, overflow: "hidden",
  },
  progressBar: {
    position: "absolute", left:0, top:0, height: "100%",
    background: "linear-gradient(90deg,#30D158,#FFD60A)",
    borderRadius: 99, transition: "width 0.3s ease",
    boxShadow: "0 0 8px #FFD60A88",
  },
  progressLabel: {
    position: "absolute", right: 6, top: -1,
    fontSize: 8, color: "#4A4A88", letterSpacing: 1, lineHeight: "10px",
    fontFamily: "'Courier New', monospace",
  },
  timerBig: {
    display:'flex', flexDirection:'column', alignItems:'center',
    background:'#0D0D25', border:'1px solid #1E1E4A',
    borderRadius:10, padding:'6px 20px', minWidth:130,
  },
  timerDigits: {
    fontSize:38, fontWeight:900, letterSpacing:4,
    fontFamily:"'Courier New',monospace", lineHeight:1,
    transition:'color 0.3s, text-shadow 0.3s',
  },
  chainBadge: {
    background: "linear-gradient(135deg,#BF5AF2,#581C87)",
    borderRadius: 6,
    padding: "4px 10px",
    fontSize: 11,
    fontWeight: 700,
    letterSpacing: 2,
    color: "#fff",
    textShadow: "0 0 10px #BF5AF2",
    boxShadow: "0 0 12px #BF5AF244",
  },

  /* board container */
  board: {
    position: "relative",
    background: "linear-gradient(180deg,#0A0A20 0%,#0E0E28 100%)",
    // subtle grid lines matching cell size
    backgroundImage: `
      linear-gradient(rgba(255,255,255,0.04) 1px, transparent 1px),
      linear-gradient(90deg, rgba(255,255,255,0.04) 1px, transparent 1px)
    `,
    backgroundSize: `${CELL}px ${CELL}px`,
    border: "2px solid #1A1A3E",
    borderRadius: 8,
    overflow: "hidden",
    boxShadow: "0 0 60px rgba(90,200,250,0.08), 0 10px 50px rgba(0,0,0,0.8)",
  },

  /* chain popup */
  chainWrap: {
    position: "absolute",
    top: "30%",
    left: "50%",
    textAlign: "center",
    pointerEvents: "none",
    zIndex: 10,
    whiteSpace: "nowrap",
  },
  chainText: {
    fontSize: 26,
    fontWeight: 900,
    color: "#FFD60A",
    textShadow: "0 0 24px #FFD60A, 0 2px 0 rgba(0,0,0,0.6)",
    letterSpacing: 2,
  },
  chainPts: {
    fontSize: 16,
    color: "#fff",
    textShadow: "0 0 10px rgba(255,255,255,0.5)",
  },

  /* overlays */
  overlay: {
    position: "absolute",
    inset: 0,
    background: "rgba(0,0,12,0.90)",
    backdropFilter: "blur(6px)",
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    zIndex: 20,
  },
  overlayCard:    { textAlign:"center", padding:"28px 36px" },
  overlayTitle:   { fontSize:34, fontWeight:900, letterSpacing:5, color:"#fff", textShadow:"0 0 30px #5AC8FA", marginBottom:10 },
  overlayScore:   { fontSize:30, fontWeight:700, letterSpacing:3, color:"#FFD60A", textShadow:"0 0 18px #FFD60A88", marginBottom:4 },
  overlayNewBest: { fontSize:13, color:"#FFD60A", letterSpacing:2, marginBottom:8, animation:"pulse 1s infinite" },
  overlayHint:    { fontSize:12, color:"#666699", marginBottom:26, letterSpacing:1, maxWidth:220 },
  overlayBtn: {
    background: "linear-gradient(135deg,#5AC8FA,#007AFF)",
    border: "none",
    borderRadius: 8,
    color: "#fff",
    fontSize: 15,
    fontWeight: 700,
    letterSpacing: 3,
    padding: "11px 28px",
    cursor: "pointer",
    boxShadow: "0 0 24px #5AC8FA66",
    fontFamily: "inherit",
    transition: "filter 0.15s",
  },

  /* control bar */
  controlBar: {
    marginTop: 10,
    display: "flex",
    alignItems: "center",
    gap: 14,
    flexWrap: "wrap",
    justifyContent: "center",
  },
  hints: { display:"flex", gap:12 },
  hint:  { fontSize:11, color:"#3A3A6A", display:"flex", alignItems:"center", gap:4 },
  kbd: {
    background: "#111130",
    border: "1px solid #222250",
    borderRadius: 3,
    padding: "1px 5px",
    fontSize: 10,
    color: "#5A5A99",
    letterSpacing: 1,
  },
  speedRow:  { display:'flex', alignItems:'center', gap:5, flexWrap:'wrap', justifyContent:'center' },
  speedLabel:{ fontSize:9, letterSpacing:2, color:'#3A3A6A', marginRight:2 },
  speedBtn:  {
    border:'none', borderRadius:5, fontSize:10, fontWeight:700,
    padding:'5px 8px', cursor:'pointer', letterSpacing:1,
    fontFamily:"'Courier New',monospace", transition:'all 0.15s',
  },
  btnRow: { display:"flex", gap:7 },
  opponentHud: {
    position: 'fixed', top: 16, right: 16, zIndex: 50,
    background: '#0D0D25', border: '1px solid #FF2D55',
    borderRadius: 10, padding: '10px 14px', textAlign: 'center',
    boxShadow: '0 0 20px #FF2D5533',
    fontFamily: "'Courier New', monospace",
  },
  opponentTitle: {
    fontSize: 9, letterSpacing: 2, color: '#FF2D55', marginBottom: 4,
  },
  btn: {
    border: "none",
    borderRadius: 6,
    color: "#fff",
    fontSize: 12,
    fontWeight: 700,
    padding: "7px 14px",
    cursor: "pointer",
    letterSpacing: 1,
    fontFamily: "inherit",
    display: "flex",
    alignItems: "center",
    gap: 5,
  },
};

/* ═══════════════════════════════════════════════════════════════
   §6  CSS KEYFRAME ANIMATIONS  (injected via <style> tag)
   ═══════════════════════════════════════════════════════════════ */
const KEYFRAMES = `
  /* ── Block flash: alternating bright/dark flicker ── */
  @keyframes cellFlash {
    0%   { opacity:1;   filter: brightness(1);   }
    15%  { opacity:0.1; filter: brightness(4.5); }
    30%  { opacity:1;   filter: brightness(2);   }
    50%  { opacity:0.1; filter: brightness(5);   }
    70%  { opacity:1;   filter: brightness(2.5); }
    85%  { opacity:0.2; filter: brightness(4);   }
    100% { opacity:0.5; filter: brightness(3);   }
  }
  .cell-flash {
    animation: cellFlash 0.13s linear infinite;
    pointer-events: none;
  }

  /* ── Cursor bracket: pulsing white glow ── */
  @keyframes cursorGlow {
    0%,100% { box-shadow: 0 0 6px #fff, 0 0 14px rgba(255,255,255,0.5); opacity:1; }
    50%     { box-shadow: 0 0 12px #fff, 0 0 28px rgba(255,255,255,0.8); opacity:0.7; }
  }
  .cursor-glow {
    animation: cursorGlow 0.65s ease-in-out infinite;
  }

  /* ── Chain popup: float upward then fade ── */
  @keyframes chainFloat {
    0%   { opacity:0; transform: translateX(-50%) scale(0.5) translateY(12px); }
    18%  { opacity:1; transform: translateX(-50%) scale(1.18) translateY(0);   }
    70%  { opacity:1; transform: translateX(-50%) scale(1)    translateY(-10px);}
    100% { opacity:0; transform: translateX(-50%) scale(0.9)  translateY(-24px);}
  }
  .chain-popup {
    animation: chainFloat 1.6s cubic-bezier(.22,1,.36,1) forwards;
  }

  /* ── Burst particle: fly outward and fade ── */
  @keyframes burst {
    0%   { opacity: 1;   transform: translate(-50%,-50%) rotate(var(--angle)) translateY(0px)   scale(1); }
    60%  { opacity: 0.8; transform: translate(-50%,-50%) rotate(var(--angle)) translateY(-22px)  scale(0.9); }
    100% { opacity: 0;   transform: translate(-50%,-50%) rotate(var(--angle)) translateY(-36px)  scale(0.3); }
  }
  .burst-particle {
    animation: burst 0.55s cubic-bezier(.2,.8,.4,1) forwards;
  }

  /* ── "New Best" pulse ── */
  @keyframes pulse {
    0%,100% { opacity:1; }
    50%     { opacity:0.5; }
  }

  * { box-sizing: border-box; }
`;
