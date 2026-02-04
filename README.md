# Chess
import React, { useState, useEffect, useRef } from 'react';

// --- Логика игры ---
const initialBoard = [
  ['br', 'bn', 'bb', 'bq', 'bk', 'bb', 'bn', 'br'],
  ['bp', 'bp', 'bp', 'bp', 'bp', 'bp', 'bp', 'bp'],
  ['', '', '', '', '', '', '', ''],
  ['', '', '', '', '', '', '', ''],
  ['', '', '', '', '', '', '', ''],
  ['', '', '', '', '', '', '', ''],
  ['wp', 'wp', 'wp', 'wp', 'wp', 'wp', 'wp', 'wp'],
  ['wr', 'wn', 'wb', 'wq', 'wk', 'wb', 'wn', 'wr'],
];

const onBoard = (r, c) => r >= 0 && r < 8 && c >= 0 && c < 8;

const getMovesForPiece = (r, c, pieceType, pieceColor, currentBoard) => {
  const moves = [];
  const opponent = pieceColor === 'w' ? 'b' : 'w';

  const checkAndAdd = (nr, nc) => {
    if (!onBoard(nr, nc)) return true;
    const target = currentBoard[nr][nc];
    if (target === '') {
      moves.push({ r: nr, c: nc });
      return false;
    } else {
      if (target[0] === opponent) moves.push({ r: nr, c: nc });
      return true;
    }
  };

  if (pieceType === 'p') {
    const direction = pieceColor === 'w' ? -1 : 1;
    const startRow = pieceColor === 'w' ? 6 : 1;
    if (onBoard(r + direction, c) && currentBoard[r + direction][c] === '') {
      moves.push({ r: r + direction, c: c });
      if (r === startRow && currentBoard[r + direction * 2][c] === '') {
        moves.push({ r: r + direction * 2, c: c });
      }
    }
    [[direction, -1], [direction, 1]].forEach(([dr, dc]) => {
      if (onBoard(r + dr, c + dc)) {
        const target = currentBoard[r + dr][c + dc];
        if (target !== '' && target[0] === opponent) moves.push({ r: r + dr, c: c + dc });
      }
    });
  }
  if (pieceType === 'n') {
    [[-2, -1], [-2, 1], [-1, -2], [-1, 2], [1, -2], [1, 2], [2, -1], [2, 1]].forEach(([dr, dc]) => checkAndAdd(r + dr, c + dc));
  }
  if (pieceType === 'k') {
    [[-1,-1], [-1,0], [-1,1], [0,-1], [0,1], [1,-1], [1,0], [1,1]].forEach(([dr, dc]) => checkAndAdd(r + dr, c + dc));
  }
  if (pieceType === 'b' || pieceType === 'q') {
    [[-1, -1], [-1, 1], [1, -1], [1, 1]].forEach(([dr, dc]) => {
      let nr = r + dr, nc = c + dc;
      while (!checkAndAdd(nr, nc)) { nr += dr; nc += dc; }
    });
  }
  if (pieceType === 'r' || pieceType === 'q') {
    [[-1, 0], [1, 0], [0, -1], [0, 1]].forEach(([dr, dc]) => {
      let nr = r + dr, nc = c + dc;
      while (!checkAndAdd(nr, nc)) { nr += dr; nc += dc; }
    });
  }
  return moves;
};

// Функция для получения всех возможных ходов определенного цвета
const getAllValidMoves = (board, color) => {
  let allMoves = [];
  board.forEach((row, r) => {
    row.forEach((cell, c) => {
      if (cell && cell[0] === color) {
        const moves = getMovesForPiece(r, c, cell[1], color, board);
        moves.forEach(m => {
          allMoves.push({
            from: { r, c },
            to: m,
            piece: cell
          });
        });
      }
    });
  });
  return allMoves;
};

// --- Компонент игры ---

export default function App() {
  const [board, setBoard] = useState(initialBoard);
  const [turn, setTurn] = useState('w');
  const [selected, setSelected] = useState(null);
  const [validMoves, setValidMoves] = useState([]);
  const [winner, setWinner] = useState(null);
  const [threeLoaded, setThreeLoaded] = useState(false);
  
  // Состояния для AI
  const [aiThinking, setAiThinking] = useState(false);
  const [batmanComment, setBatmanComment] = useState("Я слежу за каждым твоим шагом.");
  
  // Состояния для Альфреда (Подсказки)
  const [alfredHint, setAlfredHint] = useState(null);
  const [alfredThinking, setAlfredThinking] = useState(false);
  
  const mountRef = useRef(null);
  const sceneRef = useRef(null);
  const cameraRef = useRef(null);
  const rendererRef = useRef(null);
  const piecesGroupRef = useRef(null);
  const highlightsGroupRef = useRef(null);
  
  // Gemini API Key
  const apiKey = ""; // Заполняется средой выполнения

  // 1. Загрузка Three.js
  useEffect(() => {
    if (window.THREE) {
      setThreeLoaded(true);
      return;
    }
    const script = document.createElement('script');
    script.src = 'https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js';
    script.onload = () => setThreeLoaded(true);
    document.body.appendChild(script);
  }, []);

  // 2. Логика ИИ (Gemini - Бэтмен)
  useEffect(() => {
    if (turn === 'b' && !winner) {
      makeAiMove();
    }
  }, [turn, winner]);

  const makeAiMove = async () => {
    setAiThinking(true);
    setAlfredHint(null); // Скрываем подсказку Альфреда когда ход перешел
    
    const possibleMoves = getAllValidMoves(board, 'b');
    
    if (possibleMoves.length === 0) {
      setWinner('Белые');
      setAiThinking(false);
      return;
    }

    const movesListForPrompt = possibleMoves.map((m, index) => ({
      id: index,
      desc: `${m.piece} from (${m.from.r},${m.from.c}) to (${m.to.r},${m.to.c})`
    }));

    try {
      const prompt = `
        Ты играешь в шахматы за Черных фигур. Твой персонаж - Бэтмен. 
        Твой соперник (Белые) только что сделал ход.
        
        Вот список доступных тебе ходов:
        ${JSON.stringify(movesListForPrompt)}

        Твоя задача:
        1. Выбери ОДИН лучший ход (id) из этого списка, чтобы победить или захватить фигуру.
        2. Придумай короткую, мрачную фразу (комментарий) на РУССКОМ языке, обращенную к игроку.
        Это может быть угроза, философское замечание о тьме или шахматный термин.
        
        Верни ответ ТОЛЬКО в формате JSON:
        {
          "selectedMoveId": <number>,
          "comment": "<string>"
        }
      `;

      const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts: [{ text: prompt }] }],
          generationConfig: { responseMimeType: "application/json" }
        })
      });

      const data = await response.json();
      const resultText = data.candidates?.[0]?.content?.parts?.[0]?.text;
      
      if (resultText) {
        const aiResult = JSON.parse(resultText);
        const selectedMove = possibleMoves[aiResult.selectedMoveId] || possibleMoves[0];
        
        setBatmanComment(aiResult.comment);
        
        setTimeout(() => {
          executeMove(selectedMove.from.r, selectedMove.from.c, selectedMove.to.r, selectedMove.to.c);
          setAiThinking(false);
        }, 1500); // Чуть дольше задержка для чтения комментария
      } else {
        throw new Error("Empty response");
      }

    } catch (error) {
      console.error("AI Error:", error);
      const randomMove = possibleMoves[Math.floor(Math.random() * possibleMoves.length)];
      setBatmanComment("... (Тишина)");
      setTimeout(() => {
        executeMove(randomMove.from.r, randomMove.from.c, randomMove.to.r, randomMove.to.c);
        setAiThinking(false);
      }, 1000);
    }
  };

  // 3. Логика ИИ (Gemini - Альфред)
  const askAlfred = async () => {
    if (turn !== 'w' || winner || alfredThinking) return;
    
    setAlfredThinking(true);
    setAlfredHint(null);
    
    const possibleMoves = getAllValidMoves(board, 'w');
    const movesListForPrompt = possibleMoves.map((m, index) => ({
      id: index,
      desc: `${m.piece} from (${m.from.r},${m.from.c}) to (${m.to.r},${m.to.c})`
    }));

    try {
      const prompt = `
        Ты - Альфред Пенниуорт, верный дворецкий. Твой подопечный (игрок за Белых) просит совета в шахматной партии против Бэтмена.
        
        Вот доступные ходы для Белых:
        ${JSON.stringify(movesListForPrompt)}

        Твоя задача:
        1. Выбери хороший, надежный ход.
        2. Дай вежливый и краткий совет на РУССКОМ языке (максимум 1-2 предложения). Обращайся на "сэр" или "мастер Брюс" (если уместно, или просто сэр).
        
        Верни ответ ТОЛЬКО в формате JSON:
        {
          "hintText": "<string>"
        }
      `;

      const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts: [{ text: prompt }] }],
          generationConfig: { responseMimeType: "application/json" }
        })
      });

      const data = await response.json();
      const resultText = data.candidates?.[0]?.content?.parts?.[0]?.text;
      
      if (resultText) {
        const result = JSON.parse(resultText);
        setAlfredHint(result.hintText);
      }
    } catch (error) {
      console.error("Alfred Error:", error);
      setAlfredHint("Прошу прощения, сэр, связь прервалась. Я бы посоветовал развить фигуры.");
    } finally {
      setAlfredThinking(false);
    }
  };

  const executeMove = (fromR, fromC, toR, toC) => {
    setBoard(prev => {
      const newBoard = prev.map(row => [...row]);
      let movingPiece = newBoard[fromR][fromC];
      
      if (newBoard[toR][toC] && newBoard[toR][toC][1] === 'k') {
         setWinner(movingPiece[0] === 'w' ? 'Белые' : 'Черные');
      }

      newBoard[toR][toC] = movingPiece;
      newBoard[fromR][fromC] = '';

      if (movingPiece[1] === 'p') {
        if ((movingPiece[0] === 'w' && toR === 0) || (movingPiece[0] === 'b' && toR === 7)) {
          newBoard[toR][toC] = movingPiece[0] + 'q';
        }
      }
      return newBoard;
    });
    setTurn(prev => prev === 'w' ? 'b' : 'w');
  };

  // --- 3D Инициализация (остается такой же, пропущена для краткости где не менялась) ---
  useEffect(() => {
    if (!threeLoaded || !mountRef.current) return;
    if (sceneRef.current) return;

    const THREE = window.THREE;
    const width = mountRef.current.clientWidth;
    const height = mountRef.current.clientHeight;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a1a);
    sceneRef.current = scene;

    const camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000);
    camera.position.set(0, 14, 14); 
    camera.lookAt(0, 0, -2);
    cameraRef.current = camera;

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(width, height);
    renderer.shadowMap.enabled = true;
    mountRef.current.appendChild(renderer.domElement);
    rendererRef.current = renderer;

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.3);
    scene.add(ambientLight);

    const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
    dirLight.position.set(5, 10, 7);
    dirLight.castShadow = true;
    scene.add(dirLight);

    const batLight = new THREE.SpotLight(0x4444ff, 2);
    batLight.position.set(-5, 10, -5);
    batLight.target.position.set(0, 2, -6);
    batLight.castShadow = true;
    scene.add(batLight);
    scene.add(batLight.target);

    // Стол и Декор
    const tableGeo = new THREE.BoxGeometry(12, 0.5, 12);
    const tableMat = new THREE.MeshStandardMaterial({ color: 0x3d2b1f, roughness: 0.8 });
    const table = new THREE.Mesh(tableGeo, tableMat);
    table.position.y = -0.35;
    table.receiveShadow = true;
    scene.add(table);

    const legGeo = new THREE.CylinderGeometry(0.3, 0.2, 8, 8);
    [[ -5, -5], [5, -5], [-5, 5], [5, 5]].forEach(([x, z]) => {
        const leg = new THREE.Mesh(legGeo, tableMat);
        leg.position.set(x, -4.35, z);
        scene.add(leg);
    });

    // БЭТМЕН (Геометрия)
    const batmanGroup = new THREE.Group();
    const batSuitMat = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 0.4 });
    const batCapeMat = new THREE.MeshStandardMaterial({ color: 0x050505, roughness: 0.9 });
    const batSkinMat = new THREE.MeshStandardMaterial({ color: 0xeebb99, roughness: 0.4 });
    const batYellowMat = new THREE.MeshBasicMaterial({ color: 0xffd700 });
    const batEyeMat = new THREE.MeshBasicMaterial({ color: 0xffffff });

    const torso = new THREE.Mesh(new THREE.BoxGeometry(1.6, 2.0, 0.8), batSuitMat);
    torso.position.y = 1.0;
    torso.castShadow = true;
    batmanGroup.add(torso);

    const head = new THREE.Mesh(new THREE.BoxGeometry(0.7, 0.8, 0.75), batSuitMat);
    head.position.y = 2.4;
    head.castShadow = true;
    batmanGroup.add(head);

    const earGeo = new THREE.ConeGeometry(0.1, 0.4, 4);
    const leftEar = new THREE.Mesh(earGeo, batSuitMat);
    leftEar.position.set(-0.25, 2.9, 0);
    batmanGroup.add(leftEar);
    const rightEar = new THREE.Mesh(earGeo, batSuitMat);
    rightEar.position.set(0.25, 2.9, 0);
    batmanGroup.add(rightEar);

    const chin = new THREE.Mesh(new THREE.BoxGeometry(0.35, 0.25, 0.76), batSkinMat);
    chin.position.set(0, 2.15, 0);
    batmanGroup.add(chin);

    const leftEye = new THREE.Mesh(new THREE.PlaneGeometry(0.15, 0.08), batEyeMat);
    leftEye.position.set(-0.18, 2.5, 0.38);
    batmanGroup.add(leftEye);
    const rightEye = new THREE.Mesh(new THREE.PlaneGeometry(0.15, 0.08), batEyeMat);
    rightEye.position.set(0.18, 2.5, 0.38);
    batmanGroup.add(rightEye);

    const logo = new THREE.Mesh(new THREE.CylinderGeometry(0.3, 0.3, 0.1, 16), batYellowMat);
    logo.rotation.x = Math.PI / 2;
    logo.scale.x = 1.5;
    logo.position.set(0, 1.4, 0.4);
    batmanGroup.add(logo);

    const armGeo = new THREE.CylinderGeometry(0.25, 0.2, 1.4, 8);
    const leftArm = new THREE.Mesh(armGeo, batSuitMat);
    leftArm.position.set(-1.1, 1.0, 0.2);
    leftArm.rotation.z = 0.2;
    leftArm.rotation.x = -0.5;
    batmanGroup.add(leftArm);
    const rightArm = new THREE.Mesh(armGeo, batSuitMat);
    rightArm.position.set(1.1, 1.0, 0.2);
    rightArm.rotation.z = -0.2;
    rightArm.rotation.x = -0.5;
    batmanGroup.add(rightArm);

    const cape = new THREE.Mesh(new THREE.BoxGeometry(1.8, 2.5, 0.1), batCapeMat);
    cape.position.set(0, 1.0, -0.45);
    cape.rotation.x = 0.1;
    batmanGroup.add(cape);

    batmanGroup.position.set(0, -0.1, -6.5);
    scene.add(batmanGroup);

    // Стул
    const chairGroup = new THREE.Group();
    const chairBack = new THREE.Mesh(new THREE.BoxGeometry(2, 3.5, 0.2), new THREE.MeshStandardMaterial({color: 0x222222}));
    chairBack.position.set(0, 1.5, -0.6);
    chairGroup.add(chairBack);
    const chairSeat = new THREE.Mesh(new THREE.BoxGeometry(2, 0.2, 2), new THREE.MeshStandardMaterial({color: 0x222222}));
    chairSeat.position.set(0, 0, 0.4);
    chairGroup.add(chairSeat);
    chairGroup.position.set(0, -0.1, -6.5);
    scene.add(chairGroup);

    // Доска и фигуры
    const boardGroup = new THREE.Group();
    const squareGeometry = new THREE.BoxGeometry(1, 0.2, 1);
    const whiteMat = new THREE.MeshStandardMaterial({ color: 0xeeeed2, roughness: 0.5 });
    const blackMat = new THREE.MeshStandardMaterial({ color: 0x769656, roughness: 0.5 });

    for (let x = 0; x < 8; x++) {
      for (let z = 0; z < 8; z++) {
        const isWhite = (x + z) % 2 === 0;
        const square = new THREE.Mesh(squareGeometry, isWhite ? whiteMat : blackMat);
        square.position.set(x - 3.5, -0.1, z - 3.5);
        square.receiveShadow = true;
        boardGroup.add(square);
      }
    }
    scene.add(boardGroup);

    const piecesGroup = new THREE.Group();
    scene.add(piecesGroup);
    piecesGroupRef.current = piecesGroup;

    const highlightsGroup = new THREE.Group();
    scene.add(highlightsGroup);
    highlightsGroupRef.current = highlightsGroup;

    const borderGeo = new THREE.BoxGeometry(8.5, 0.2, 8.5);
    const borderMat = new THREE.MeshStandardMaterial({ color: 0x5c4033 });
    const border = new THREE.Mesh(borderGeo, borderMat);
    border.position.y = -0.2;
    border.receiveShadow = true;
    scene.add(border);

    const animate = () => {
      requestAnimationFrame(animate);
      renderer.render(scene, camera);
    };
    animate();

    const handleResize = () => {
      if (mountRef.current && cameraRef.current && rendererRef.current) {
         const w = mountRef.current.clientWidth;
         const h = mountRef.current.clientHeight;
         cameraRef.current.aspect = w / h;
         cameraRef.current.updateProjectionMatrix();
         rendererRef.current.setSize(w, h);
      }
    };
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, [threeLoaded]);

  // --- Геометрия фигур (Без изменений) ---
  const createPieceMesh = (type, color) => {
    const THREE = window.THREE;
    const material = new THREE.MeshStandardMaterial({
      color: color === 'w' ? 0xffffff : 0x222222,
      roughness: 0.2,
      metalness: 0.3
    });

    const group = new THREE.Group();
    const baseGeo = new THREE.CylinderGeometry(0.35, 0.35, 0.1, 32);
    const base = new THREE.Mesh(baseGeo, material);
    base.position.y = 0.05;
    base.castShadow = true;
    group.add(base);

    if (type === 'p') {
      const bodyGeo = new THREE.ConeGeometry(0.25, 0.6, 16);
      const body = new THREE.Mesh(bodyGeo, material);
      body.position.y = 0.35;
      body.castShadow = true;
      group.add(body);
      const headGeo = new THREE.SphereGeometry(0.15, 16, 16);
      const head = new THREE.Mesh(headGeo, material);
      head.position.y = 0.65;
      head.castShadow = true;
      group.add(head);
    } else if (type === 'r') {
      const bodyGeo = new THREE.CylinderGeometry(0.3, 0.3, 0.6, 16);
      const body = new THREE.Mesh(bodyGeo, material);
      body.position.y = 0.35;
      body.castShadow = true;
      group.add(body);
      const topGeo = new THREE.CylinderGeometry(0.35, 0.35, 0.15, 6);
      const top = new THREE.Mesh(topGeo, material);
      top.position.y = 0.7;
      top.castShadow = true;
      group.add(top);
    } else if (type === 'n') {
      const bodyGeo = new THREE.CylinderGeometry(0.25, 0.3, 0.4, 16);
      const body = new THREE.Mesh(bodyGeo, material);
      body.position.y = 0.25;
      group.add(body);
      const headGeo = new THREE.BoxGeometry(0.2, 0.35, 0.4);
      const head = new THREE.Mesh(headGeo, material);
      head.position.set(0, 0.55, 0);
      head.rotation.x = -0.2;
      head.castShadow = true;
      group.add(head);
    } else if (type === 'b') {
      const bodyGeo = new THREE.CylinderGeometry(0.15, 0.3, 0.7, 16);
      const body = new THREE.Mesh(bodyGeo, material);
      body.position.y = 0.4;
      group.add(body);
      const headGeo = new THREE.CylinderGeometry(0.05, 0.2, 0.2, 16);
      const head = new THREE.Mesh(headGeo, material);
      head.position.y = 0.8;
      head.castShadow = true;
      group.add(head);
    } else if (type === 'q') {
      const bodyGeo = new THREE.CylinderGeometry(0.2, 0.35, 0.9, 16);
      const body = new THREE.Mesh(bodyGeo, material);
      body.position.y = 0.5;
      group.add(body);
      const crownGeo = new THREE.TorusGeometry(0.15, 0.05, 8, 16);
      const crown = new THREE.Mesh(crownGeo, material);
      crown.position.y = 1.0;
      crown.rotation.x = Math.PI / 2;
      crown.castShadow = true;
      group.add(crown);
      const ballGeo = new THREE.SphereGeometry(0.1, 16, 16);
      const ball = new THREE.Mesh(ballGeo, material);
      ball.position.y = 1.0;
      group.add(ball);
    } else if (type === 'k') {
      const bodyGeo = new THREE.CylinderGeometry(0.25, 0.35, 1.0, 16);
      const body = new THREE.Mesh(bodyGeo, material);
      body.position.y = 0.55;
      group.add(body);
      const crossV = new THREE.BoxGeometry(0.1, 0.3, 0.1);
      const crossV_m = new THREE.Mesh(crossV, material);
      crossV_m.position.y = 1.2;
      group.add(crossV_m);
      const crossH = new THREE.BoxGeometry(0.25, 0.1, 0.1);
      const crossH_m = new THREE.Mesh(crossH, material);
      crossH_m.position.y = 1.2;
      group.add(crossH_m);
    }

    if (color === 'b') group.rotation.y = Math.PI;
    if (color === 'w') group.rotation.y = 0;
    if (type === 'n') {
       if (color === 'w') group.rotation.y = Math.PI; 
       if (color === 'b') group.rotation.y = 0;
    }
    return group;
  };

  // --- Синхронизация ---
  useEffect(() => {
    if (!threeLoaded || !piecesGroupRef.current) return;
    const THREE = window.THREE;
    const group = piecesGroupRef.current;

    while(group.children.length > 0){ 
        group.remove(group.children[0]); 
    }

    board.forEach((row, r) => {
      row.forEach((cell, c) => {
        if (cell) {
          const color = cell[0];
          const type = cell[1];
          const mesh = createPieceMesh(type, color);
          mesh.position.set(c - 3.5, 0, r - 3.5);
          group.add(mesh);
        }
      });
    });
  }, [board, threeLoaded]);

  useEffect(() => {
    if (!threeLoaded || !highlightsGroupRef.current) return;
    const THREE = window.THREE;
    const group = highlightsGroupRef.current;
    
    while(group.children.length > 0) group.remove(group.children[0]);

    if (selected) {
      const selGeo = new THREE.PlaneGeometry(0.9, 0.9);
      const selMat = new THREE.MeshBasicMaterial({ color: 0xffff00, transparent: true, opacity: 0.5, side: THREE.DoubleSide });
      const selMesh = new THREE.Mesh(selGeo, selMat);
      selMesh.rotation.x = -Math.PI / 2;
      selMesh.position.set(selected.c - 3.5, 0.01, selected.r - 3.5);
      group.add(selMesh);
    }

    validMoves.forEach(move => {
      const moveGeo = new THREE.CircleGeometry(0.15, 32);
      const moveMat = new THREE.MeshBasicMaterial({ color: 0x00ff00, transparent: true, opacity: 0.8, side: THREE.DoubleSide });
      const moveMesh = new THREE.Mesh(moveGeo, moveMat);
      moveMesh.rotation.x = -Math.PI / 2;
      moveMesh.position.set(move.c - 3.5, 0.02, move.r - 3.5);
      
      if (board[move.r][move.c]) {
          moveMesh.material.color.setHex(0xff0000);
          moveMesh.scale.set(2, 2, 2);
          moveMesh.geometry = new THREE.RingGeometry(0.3, 0.4, 32);
      }
      group.add(moveMesh);
    });
  }, [selected, validMoves, threeLoaded, board]);

  // Обработка клика
  const handleCanvasClick = (event) => {
    if (!threeLoaded || winner || turn === 'b' || aiThinking) return;
    
    const THREE = window.THREE;
    const rect = mountRef.current.getBoundingClientRect();
    const x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    const y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

    const raycaster = new THREE.Raycaster();
    raycaster.setFromCamera({ x, y }, cameraRef.current);

    const plane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
    const target = new THREE.Vector3();
    raycaster.ray.intersectPlane(plane, target);

    if (target) {
        const c = Math.round(target.x + 3.5);
        const r = Math.round(target.z + 3.5);

        if (onBoard(r, c)) {
            logicClick(r, c);
        }
    }
  };

  const logicClick = (r, c) => {
    const piece = board[r][c];
    const isMyPiece = piece && piece[0] === 'w';

    if (isMyPiece) {
      if (selected && selected.r === r && selected.c === c) {
        setSelected(null);
        setValidMoves([]);
      } else {
        setSelected({ r, c });
        setValidMoves(getMovesForPiece(r, c, piece[1], turn, board));
      }
      return;
    }

    if (selected) {
      const isMoveValid = validMoves.some(m => m.r === r && m.c === c);
      if (isMoveValid) {
        executeMove(selected.r, selected.c, r, c);
        setSelected(null);
        setValidMoves([]);
      } else {
        setSelected(null);
        setValidMoves([]);
      }
    }
  };

  const resetGame = () => {
    setBoard(initialBoard);
    setTurn('w');
    setSelected(null);
    setValidMoves([]);
    setWinner(null);
    setBatmanComment("Готов проиграть?");
    setAlfredHint(null);
  };

  const rotateBoard = (dir) => {
      if(!sceneRef.current) return;
      const cam = cameraRef.current;
      const x = cam.position.x;
      const z = cam.position.z;
      const angle = dir * 0.1;
      cam.position.x = x * Math.cos(angle) - z * Math.sin(angle);
      cam.position.z = x * Math.sin(angle) + z * Math.cos(angle);
      cam.lookAt(0,0,0);
  };

  return (
    <div className="w-full h-screen bg-neutral-900 flex flex-col relative overflow-hidden font-sans">
      
      {/* UI Overlay */}
      <div className="absolute top-4 left-0 right-0 z-10 flex flex-col items-center pointer-events-none">
        <h1 className="text-3xl font-bold text-white drop-shadow-lg mb-2 tracking-widest uppercase">Gotham Chess ✨</h1>
        <div className="flex gap-4 pointer-events-auto">
             <div className={`px-4 py-1 rounded shadow transition-all ${turn === 'w' ? 'bg-white text-black scale-110 border-2 border-yellow-400' : 'bg-gray-700 text-gray-400'}`}>
                Вы
             </div>
             <div className={`px-4 py-1 rounded shadow transition-all ${turn === 'b' ? 'bg-black text-white scale-110 border-2 border-yellow-400' : 'bg-gray-700 text-gray-400'}`}>
                {aiThinking ? 'Думает...' : 'Бэтмен'}
             </div>
        </div>
      </div>

      {/* Comic Bubble for Batman */}
      <div className="absolute top-[20%] left-1/2 transform -translate-x-1/2 pointer-events-none z-20 w-80">
         <div className="relative bg-white text-black p-4 rounded-2xl shadow-xl border-2 border-black animate-fade-in-up">
            <p className="font-bold text-center text-sm uppercase">{batmanComment}</p>
            <div className="absolute bottom-[-10px] left-1/2 transform -translate-x-1/2 w-4 h-4 bg-white border-r-2 border-b-2 border-black rotate-45"></div>
         </div>
      </div>

      {/* Alfred's Hint Bubble */}
      {alfredHint && (
        <div className="absolute bottom-[20%] left-8 pointer-events-auto z-20 w-64">
           <div className="relative bg-neutral-200 text-neutral-900 p-4 rounded-xl shadow-2xl border border-neutral-400 animate-fade-in">
              <h4 className="text-xs font-bold uppercase text-neutral-600 mb-1">Альфред Пенниуорт</h4>
              <p className="text-sm italic">"{alfredHint}"</p>
              <button 
                onClick={() => setAlfredHint(null)}
                className="absolute top-1 right-2 text-neutral-400 hover:text-black"
              >✕</button>
           </div>
        </div>
      )}

      {/* 3D Canvas */}
      <div 
        ref={mountRef} 
        className="flex-1 w-full h-full cursor-pointer"
        onClick={handleCanvasClick}
      >
        {!threeLoaded && (
            <div className="absolute inset-0 flex items-center justify-center text-white">
                Загрузка 3D движка...
            </div>
        )}
      </div>

      {/* Bottom Controls */}
      <div className="absolute bottom-8 left-0 right-0 z-10 flex flex-col items-center gap-4 pointer-events-none">
         {winner && (
            <div className="bg-black/90 text-white p-6 rounded-xl backdrop-blur-md pointer-events-auto text-center border-2 border-yellow-500 animate-bounce shadow-[0_0_20px_rgba(255,215,0,0.5)]">
              <h2 className="text-2xl font-bold text-yellow-400 mb-2">
                  {winner === 'Белые' ? 'ВЫ ПОБЕДИЛИ БЭТМЕНА!' : 'БЭТМЕН ПОБЕДИЛ!'}
              </h2>
              <button onClick={resetGame} className="mt-4 bg-yellow-600 hover:bg-yellow-500 text-black font-bold py-2 px-6 rounded-full transition">
                Реванш
              </button>
            </div>
         )}

         {!winner && (
             <div className="flex gap-4 pointer-events-auto items-center">
                <button onClick={() => rotateBoard(1)} className="bg-neutral-800 hover:bg-neutral-700 text-white p-3 rounded-full shadow-lg border border-gray-600" title="Повернуть влево">
                    ↺
                </button>
                
                <div className="flex gap-2">
                    <button 
                        onClick={askAlfred} 
                        disabled={turn !== 'w' || alfredThinking}
                        className={`px-4 py-2 rounded-full font-bold shadow-lg border border-blue-900 transition flex items-center gap-2 ${turn === 'w' && !alfredThinking ? 'bg-blue-800 hover:bg-blue-700 text-white' : 'bg-neutral-700 text-neutral-500 cursor-not-allowed'}`}
                    >
                        {alfredThinking ? 'Спрашиваю...' : '✨ Совет Альфреда'}
                    </button>
                    
                    <button onClick={resetGame} className="bg-red-900/80 hover:bg-red-800 text-white px-4 py-2 rounded-full font-bold shadow-lg border border-red-900">
                        Сдаться
                    </button>
                </div>

                <button onClick={() => rotateBoard(-1)} className="bg-neutral-800 hover:bg-neutral-700 text-white p-3 rounded-full shadow-lg border border-gray-600" title="Повернуть вправо">
                    ↻
                </button>
             </div>
         )}
      </div>

    </div>
  );
}
