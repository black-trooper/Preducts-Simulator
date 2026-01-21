<script>
  import { onMount, tick } from "svelte";
  import Tips from "./Tips.svelte";
  import DraggableArea from "./DraggableArea.svelte";
  import Module from "./Module.svelte";

  const currentVersion = "0.3.0";

  let moduleIdCounter = 0; // モジュールID用カウンター
  let deskOptions = [];
  let selectedDesk = null;
  let modules = [];
  let placedModules = [];
  let localStorageKey = "PreductsSimulator";
  let scale = 1;
  let paddingWidth = 20;
  let paddingHeight = 13;

  // 現在のアプリバージョン

  onMount(async () => {
    await loadDeskOptions();
    await loadModules();

    loadData();

    // スケールを計算
    calculateScale();
    window.addEventListener("resize", calculateScale);
  });

  const calculateScale = () => {
    // 挙動不審なので一旦無効
    if (!selectedDesk) return;
    // デスクとダッシュボードに応じたpaddingを設定
    if (selectedDesk.type == "desk") {
      paddingWidth = 20;
      paddingHeight = 13;
    } else {
      paddingWidth = 60;
      paddingHeight = 50;
    }
    const maxWidth = 1200 - (paddingWidth + 1) * 2;
    const marginWidth = paddingWidth * 2 + 17;
    const screenWidth =
      window.innerWidth - marginWidth < maxWidth
        ? window.innerWidth - marginWidth
        : maxWidth;
    const screenHeight = window.innerHeight;
    const scaleX = screenWidth / selectedDesk.width;
    const scaleY = screenHeight / selectedDesk.height;
    // デスク全体が収まるようにスケールを設定
    scale = Math.min(scaleX, scaleY, 1); // 最大スケールを1（等倍）に制限
  };

  const loadDeskOptions = async () => {
    try {
      const response = await fetch("/desks.json");
      const rawDesks = await response.json();

      deskOptions = rawDesks.map((desk) => {
        const rails = expandRails(desk.railPattern);
        const holes = expandHoles(desk.holePattern || []);
        const beams = expandBeams(desk.beamPattern || []);

        return {
          ...desk,
          rails,
          holes,
          beams,
        };
      });
    } catch (error) {
      console.error("データの読み込みに失敗しました", error);
    }
  };

  const expandRails = (pattern) => {
    if (!pattern) return [];

    const stepY = pattern.steps.y || [];
    const rows = stepY.length + 1;

    const rails = [];

    for (let row = 0; row < rows; row++) {
      const y =
        pattern.origin.y + stepY.slice(0, row).reduce((a, v) => a + v, 0);
      rails.push({
        x: pattern.origin.x,
        y,
        length: pattern.length,
      });
    }
    return rails;
  };

  const expandHoles = (patterns) => {
    const holes = [];

    patterns.forEach((pattern) => {
      const steps = pattern.steps || {};
      const stepX = steps.x || [];
      const stepY = steps.y || [];

      const cols = stepX.length + 1;
      const rows = stepY.length + 1;
      for (let row = 0; row < rows; row++) {
        const y =
          pattern.origin.y + stepY.slice(0, row).reduce((a, v) => a + v, 0);

        for (let col = 0; col < cols; col++) {
          const x =
            pattern.origin.x + stepX.slice(0, col).reduce((a, v) => a + v, 0);

          holes.push({
            x,
            y,
            diameter: pattern.diameter,
          });
        }
      }
    });
    return holes;
  };

  const expandBeams = (patterns) => {
    if (!Array.isArray(patterns)) return [];

    return patterns.flatMap((pattern) => {
      const xArray = Array.isArray(pattern.x) ? pattern.x : [pattern.x];
      return xArray.map((xVal) => ({
        x: xVal,
        y: pattern.y,
        length: pattern.length,
        height: pattern.height,
        notCross: pattern.notCross,
      }));
    });
  };

  const loadModules = async () => {
    try {
      const response = await fetch("/modules.json?v=" + currentVersion);
      if (!response.ok) {
        throw new Error("モジュールのデータの取得に失敗しました");
      }

      const rawModules = await response.json();
      // holeGrid を holes に展開
      for (const module of rawModules) {
        for (const install of module.install) {
          if (install.holeGrid) {
            install.holes = expandHoleGrid(install.holeGrid);
            delete install.holeGrid; // 不要なら削除
          }
        }
      }
      modules = rawModules.map((module) => {
        module = {
          ...module,
          width: module.install[0].width,
          height: module.install[0].height,
          holes: module.install[0].holes,
          currentSizeIndex: 0, // 現在のサイズインデックスを更新
        };
        return module;
      });
    } catch (error) {
      console.error("エラー:", error);
    }
  };

  const expandHoleGrid = (holeGrid) => {
    const origin = holeGrid.origin;
    const size = holeGrid.size;
    const steps = holeGrid.steps || {};

    const stepX = steps.x || [];
    const stepY = steps.y || [];

    const cols = stepX.length + 1;
    const rows = stepY.length + 1;

    const holes = [];

    for (let row = 0; row < rows; row++) {
      const y = origin.y + stepY.slice(0, row).reduce((a, v) => a + v, 0);

      for (let col = 0; col < cols; col++) {
        const x = origin.x + stepX.slice(0, col).reduce((a, v) => a + v, 0);

        holes.push({
          x,
          y,
          width: size.width,
          height: size.height,
        });
      }
    }

    return holes;
  };

  // デスク選択
  const selectDesk = (desk) => {
    selectedDesk = desk;
    placedModules = []; // デスク変更時にリセット
    calculateScale();
    saveData();
  };

  // 再選択機能を追加
  const resetDesk = () => {
    selectedDesk = null;
    placedModules = [];
    saveData();
  };

  // モジュール配置処理を修正
  const placeModule = (moduleId) => {
    if (!selectedDesk) return;

    // モジュールのサイズ情報を取得
    const module = modules.find((m) => m.id === moduleId);
    if (!module) return;

    placedModules = [
      ...placedModules,
      {
        ...module,
        id: moduleIdCounter++,
        x: 0,
        y: 0,
      },
    ];
    saveData();
  };

  const updateModulePosition = (index, newPosition) => {
    // すでにモジュールは移動済みなので衝突判定の情報を更新しているだけ
    placedModules = placedModules.map((module, i) => {
      if (i === index) {
        return { ...module, x: newPosition.x, y: newPosition.y };
      }
      return module;
    });
    updateCollisionStates();
    saveData();
  };

  const updateCollisionStates = () => {
    placedModules.forEach((module) => {
      module.isValid = validateModulePlacement(module);
    });
  };

  const checkHolesAndRails = (module) => {
    const getDistance = (hole1, hole2) => {
      return Math.sqrt(
        Math.pow(hole1.x - hole2.x, 2) + Math.pow(hole1.y - hole2.y, 2),
      );
    };

    const getClosestHolePairs = (holes) => {
      // 穴同士の距離を計算し、近いペアを返す
      const pairs = [];
      for (let i = 0; i < holes.length; i++) {
        for (let j = i + 1; j < holes.length; j++) {
          const dist = getDistance(holes[i], holes[j]);
          pairs.push({ hole1: holes[i], hole2: holes[j], distance: dist });
        }
      }

      // 距離が最も小さいペアを選ぶ
      const sortedPairs = pairs.sort((a, b) => a.distance - b.distance);
      return sortedPairs.slice(0, holes.length / 2);
    };

    const isValidForRails = (hole) => {
      const holeGlobal = {
        x: module.x + hole.x,
        y: module.y + hole.y,
        width: hole.width,
        height: hole.height,
      };

      const matchesRail = selectedDesk.rails.some((rail) => {
        const railBounds = {
          x: rail.x,
          y: rail.y,
          width: rail.length,
          height: 5, // レールの高さ
        };

        // レールと穴が交差するかどうか
        return (
          holeGlobal.x >= railBounds.x &&
          holeGlobal.x + holeGlobal.width <= railBounds.x + railBounds.width &&
          holeGlobal.y <= railBounds.y &&
          holeGlobal.y + holeGlobal.height >= railBounds.y + railBounds.height
        );
      });
      const matchesDeskHole = selectedDesk.holes.some((deskHole) => {
        const deskHoleGlobal = {
          x: deskHole.x,
          y: deskHole.y,
          width: deskHole.diameter,
          height: deskHole.diameter,
        };

        // デスクのネジ穴がモジュールのネジ穴に完全に収まるか
        return (
          deskHoleGlobal.x >= holeGlobal.x &&
          deskHoleGlobal.x + deskHoleGlobal.width <=
            holeGlobal.x + holeGlobal.width &&
          deskHoleGlobal.y >= holeGlobal.y &&
          deskHoleGlobal.y + deskHoleGlobal.height <=
            holeGlobal.y + holeGlobal.height
        );
      });
      return matchesRail || matchesDeskHole;
    };

    return module.holes.every((hole, index) => {
      // 8つ以上の穴の場合はペアを作成
      if (module.holes.length >= 8) {
        const closestPairs = getClosestHolePairs(module.holes);

        // ペアで分けた穴がレールにヒットするかを判定
        const hits = closestPairs.every((pair) => {
          return isValidForRails(pair.hole1) || isValidForRails(pair.hole2);
        });

        return hits;
      }

      return isValidForRails(hole);
    });
  };

  // モジュール同士の衝突判定を追加
  const checkModuleCollision = (module) => {
    return placedModules.some((otherModule) => {
      if (module.id === otherModule.id) {
        return false; // 自分自身との衝突は無視
      }

      // 矩形が重なっているか判定
      return (
        module.x < otherModule.x + otherModule.width &&
        module.x + module.width > otherModule.x &&
        module.y < otherModule.y + otherModule.height &&
        module.y + module.height > otherModule.y
      );
    });
  };

  // ビームとの衝突判定
  const checkBeamCollision = (module) => {
    for (let beam of selectedDesk.beams) {
      const overlaps =
        module.x < beam.x + beam.length &&
        module.x + module.width > beam.x &&
        module.y < beam.y + beam.height &&
        module.y + module.height > beam.y;

      if (!overlaps) continue;

      // crossBeam をまたげるが、notCross ビームならブロックする
      if (module.crossBeam && !beam.notCross) {
        continue; // 通過OK
      }

      // それ以外は衝突として扱う
      return true;
    }

    return false; // 衝突なし
  };

  // 条件を統合してモジュールを検証
  const validateModulePlacement = (module) => {
    return (
      checkHolesAndRails(module) &&
      !checkModuleCollision(module) &&
      !checkBeamCollision(module)
    );
  };

  // データ保存
  const saveData = () => {
    const data = {
      desk: selectedDesk,
      modules: placedModules,
    };
    localStorage.setItem(localStorageKey, JSON.stringify(data));
  };

  // データ読み込み
  const loadData = () => {
    // ローカルストレージからデータを読み込み
    const savedVersion = localStorage.getItem("app_version");
    const savedData = localStorage.getItem(localStorageKey);

    // バージョンが異なる場合にローカルストレージを削除
    if (savedVersion !== currentVersion) {
      localStorage.clear();
      localStorage.setItem("app_version", currentVersion);
    }

    if (savedData) {
      placedModules = [];
      const { desk, modules } = JSON.parse(savedData);
      selectedDesk = desk;
      placedModules = modules;
      moduleIdCounter = Math.max(...modules.map((module) => module.id)) + 1;
    }
  };

  // モジュール削除
  const removeModule = async (index) => {
    // リフレッシュさせるため一旦クリアしている
    const tempModules = structuredClone(
      placedModules.filter((_, i) => i !== index),
    );
    placedModules = [];
    await tick();
    placedModules = tempModules;
    updateCollisionStates();
    saveData();
  };

  const clearModules = () => {
    placedModules = [];
    saveData();
  };
</script>

<div class="container">
  {#if !selectedDesk}
    <div class="hero">
      <img src="../hero.png" alt="PREDUCTS Module Simulator" />
      <h1>PREDUCTS Module Simulator</h1>
      {currentVersion}
    </div>
    <div class="description">
      <p>
        PREDUCTSのモジュール配置をシミュレーションできる非公式のツールです。
      </p>
      <p>
        デスクやモジュールのサイズは公式情報を元にしていますが、目分量で設定している箇所もあるため誤差があります。
      </p>
      <p>
        不具合に気づいた場合や、モジュールの追加要望は <a
          href="https://github.com/black-trooper/PREDUCTS-Module-Simulator"
          >Github</a
        > にて Pull Request をいただけるととても助かります。
      </p>
    </div>
    <hr />
    <div class="palette">
      <div class="category">
        <h4>DESK</h4>
        <div class="module-buttons">
          {#each deskOptions.filter((desk) => desk.type === "desk") as desk}
            <button on:click={() => selectDesk(desk)}>
              {desk.name}
            </button>
          {/each}
        </div>
      </div>

      <div class="category">
        <h4>DASHBOARD</h4>
        <div class="module-buttons">
          {#each deskOptions.filter((desk) => desk.type === "dashboard") as dashboard}
            <button on:click={() => selectDesk(dashboard)}>
              {dashboard.name}
            </button>
          {/each}
          {#if deskOptions.filter((desk) => desk.type === "dashboard").length === 0}
            <div style="text-align:center; margin-top:3rem">Coming soon</div>
          {/if}
        </div>
      </div>
    </div>
  {:else}
    <div class="header">
      <h3 class="header-title">{selectedDesk.name}</h3>
      <div class="header-buttons">
        <Tips></Tips>
        <button class="reset-btn" on:click={clearModules}>Clear Modules</button>
        <button class="reset-btn" on:click={resetDesk}>Change Desk</button>
      </div>
    </div>
    <div
      style="padding:{paddingHeight}px {paddingWidth}px;height:{selectedDesk.height *
        scale +
        20}px;overflow:visible;"
    >
      <DraggableArea>
        <div
          class="desk-area"
          style="
          width: {selectedDesk.width}px;
          height: {selectedDesk.height}px;
          transform: scale({scale});
          transform-origin: top left;"
        >
          <div class="front-line"></div>
          <!-- デスクのレールを表示 -->
          {#each selectedDesk.rails as rail}
            <div
              class="rail"
              style=" top: {rail.y - 2.5}px;
                      left: {rail.x}px;
                      width: {rail.length}px;"
            ></div>
          {/each}
          <!-- デスクのネジ穴を表示 -->
          {#each selectedDesk.holes as hole}
            <div
              class="hole"
              style=" top: {hole.y - hole.diameter / 2}px;
                      left: {hole.x - hole.diameter / 2}px;
                      width: {hole.diameter}px;
                      height: {hole.diameter}px;"
            ></div>
          {/each}
          <!-- デスクのビームを表示 -->
          <!-- 縦長のビーム（左右）を先に描画 -->
          {#each selectedDesk.beams.filter((b) => b.height > b.length) as beam}
            {@const isLeftBeam = beam.x < selectedDesk.width / 2}
            {@const borderRadius = isLeftBeam
              ? "border-top-right-radius: 8px; border-bottom-right-radius: 8px;"
              : "border-top-left-radius: 8px; border-bottom-left-radius: 8px;"}
            {@const opacity = beam.notCross ? "opacity: 0;" : ""}
            <div
              class="beam {isLeftBeam ? 'beam-left' : 'beam-right'}"
              style="top: {beam.y}px; left: {beam.x}px; width: {beam.length}px; height: {beam.height}px; {borderRadius} {opacity}"
            ></div>
          {/each}
          <!-- 横長のビーム（中央）を後から描画して上に重ねる -->
          {#each selectedDesk.beams.filter((b) => b.length >= b.height) as beam}
            {@const opacity = beam.notCross ? "opacity: 0;" : ""}
            <div
              class="beam beam-horizontal"
              style="top: {beam.y}px; left: {beam.x +
                3}px; width: {beam.length -
                6}px; height: {beam.height}px; {opacity}"
            ></div>
          {/each}

          {#each placedModules as module, index}
            <Module
              {scale}
              {module}
              {index}
              {selectedDesk}
              updatePosition={updateModulePosition}
              {updateCollisionStates}
              {removeModule}
            />
          {/each}
        </div>
      </DraggableArea>
    </div>
    <!-- モジュールリストの表示 -->
    <div class="palette">
      <div class="category">
        <h4>Module / Hanger</h4>
        <div class="module-buttons">
          {#each modules.filter((module) => module.type === "mount" && module[selectedDesk.type]) as mountModule}
            <button on:click={() => placeModule(mountModule.id)}>
              {mountModule.name}
            </button>
          {/each}
        </div>
      </div>

      <div class="category">
        <h4>Tray / Drawer</h4>
        <div class="module-buttons">
          {#each modules.filter((module) => module.type === "tray" && module[selectedDesk.type]) as trayModule}
            <button on:click={() => placeModule(trayModule.id)}>
              {trayModule.name}
            </button>
          {/each}
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  .container {
    max-width: 1200px;
    margin: auto;
  }

  .hero {
    position: relative;
    width: 100%;
    height: auto; /* 高さを自動調整 */
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    color: white;
    overflow: hidden;
  }

  .hero img {
    width: 100%;
    height: auto;
    opacity: 0.4; /* 透明度調整 */
  }

  .hero h1 {
    position: absolute;
    padding: 10px;
    border-radius: 8px;
    font-size: 2em;
    color: black;
  }

  @media (max-width: 768px) {
    .hero h1 {
      font-size: 1.5em; /* モバイル用にフォントサイズ調整 */
    }
  }
  hr {
    height: 0;
    padding: 0;
    border: 0;
    margin: 10px;
    border-top: 1px solid rgba(0, 0, 0, 0.1);
  }
  .description {
    padding: 10px;
  }
  .header {
    display: flex;
    justify-content: space-between; /* 両端に配置 */
    align-items: center; /* 垂直方向に中央揃え */
    margin: 0;
    padding: 0 10px;
  }

  .header-title {
    margin: 0; /* 余計な余白を排除 */
    padding-bottom: 2px;
  }

  .header-buttons {
    display: flex;
    gap: 10px; /* ボタン間に適度な間隔を設定 */
  }
  .reset-btn {
    margin: 0;
  }
  .desk-area {
    position: relative;
    border: 1px solid #444;
    margin: 10px auto;
    background-color: #d9d9d9;
  }
  .front-line {
    position: absolute;
    width: 100%;
    height: 7px;
    top: 0;
    background-color: #b0b0b0;
  }
  .rail {
    position: absolute;
    height: 5px;
    background-color: #ccc;
    border: 1px solid #333;
  }
  .hole {
    position: absolute;
    background-color: #333;
    border: 1px solid #222;
    border-radius: 50%;
  }
  .hole::before {
    content: "";
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 200%;
    height: 200%;
    border: 1px solid #222;
    border-radius: 50%;
    background-color: transparent;
  }
  .beam {
    position: absolute;
    background-color: #b3b3b3;
    border: 1px solid #222;
  }
  .beam-left::before,
  .beam-right::before {
    content: "";
    position: absolute;
    top: 0;
    bottom: 0;
    width: 1px;
    background-color: #222;
  }
  .beam-left::before {
    left: 2px;
  }
  .beam-right::before {
    right: 2px;
  }
  .beam-horizontal {
    border: none;
    border-top: 1px solid #222;
    border-bottom: 1px solid #222;
    border-left: 1px solid #222;
    background-image: linear-gradient(to right, #222 0, #222 100%),
      linear-gradient(to right, #222 0, #222 100%);
    background-size:
      100% 1px,
      100% 1px;
    background-position:
      0 20px,
      0 calc(100% - 20px);
    background-repeat: no-repeat;
  }
  .palette {
    display: flex;
    flex-direction: row;
    gap: 20px; /* カテゴリ間の余白 */
    justify-content: center; /* 中央揃え */
    padding: 0 10px 4rem 10px;
  }

  .category {
    flex: 1; /* 各カテゴリの幅を均等にする */
    display: flex;
    flex-direction: column;
    gap: 10px; /* 見出しとボタン群の余白 */
  }

  .category h4 {
    font-size: 1.2rem;
    text-align: center;
    margin-bottom: 10px;
  }

  .module-buttons {
    display: grid;
    grid-template-columns: repeat(
      auto-fit,
      minmax(200px, 1fr)
    ); /* 最小200px、可能なら均等に分割 */
    gap: 10px; /* ボタン間のスペース */
    z-index: 100;
  }
</style>
