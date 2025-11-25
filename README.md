<!doctype html>
<html lang="ja">
  <head>
    <meta charset="utf-8" />
    <title>画像分割＆透過ツール</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />

    <!-- Tailwind 設定（アニメーション用だけ軽く拡張） -->
    <script>
      tailwind = window.tailwind || {};
      tailwind.config = {
        theme: {
          extend: {
            keyframes: {
              fadeIn: {
                "0%": { opacity: 0 },
                "100%": { opacity: 1 },
              },
            },
            animation: {
              fadeIn: "fadeIn 0.2s ease-out",
            },
          },
        },
      };
    </script>

    <!-- Tailwind CSS（CDN版） -->
    <script src="https://cdn.tailwindcss.com"></script>

    <!-- React / ReactDOM（CDN版） -->
    <script crossorigin src="https://unpkg.com/react@18/umd/react.development.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>

    <!-- lucide-react（ReactアイコンのCDN版 UMD） -->
    <script src="https://unpkg.com/lucide-react@0.339.0/dist/umd/lucide-react.min.js"></script>

    <!-- Babel（JSX をブラウザ側で変換） -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  </head>

  <body class="bg-gray-50">
    <div id="root"></div>

    <!-- React コード本体 -->
    <script type="text/babel" data-presets="env,react">
      const { useState, useRef, useEffect, useCallback } = React;
      const {
        Upload,
        X,
        Download,
        Settings,
        RefreshCw,
        Check,
        Image: ImageIcon,
        MousePointer2,
        Move,
        Maximize2,
        Plus,
        Trash2,
        Grid,
        Layers,
        Save,
        Info,
        ArrowRight,
      } = lucideReact;

      // JSZipの読み込み
      const LoadJSZip = ({ onLoad }) => {
        useEffect(() => {
          if (window.JSZip) {
            onLoad();
            return;
          }
          const script = document.createElement("script");
          script.src =
            "https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js";
          script.onload = onLoad;
          document.body.appendChild(script);
        }, [onLoad]);
        return null;
      };

      // 推奨サイズ定数
      const PRESET_SIZES = [
        { label: "メイン画像", w: 240, h: 240, desc: "1個 (必須)" },
        { label: "スタンプ画像", w: 370, h: 320, desc: "8~40個 (最大)" },
        { label: "トークルームタブ", w: 96, h: 74, desc: "1個 (必須)" },
      ];

      const ImageSplitter = () => {
        const [image, setImage] = useState(null);
        const [imageUrl, setImageUrl] = useState(null);
        const [jsZipLoaded, setJsZipLoaded] = useState(false);

        // 設定
        const [settings, setSettings] = useState({
          cols: 4,
          rows: 3,
          enableTransparent: true,
          targetColor: "#FFFFFF",
          tolerance: 10,
          mode: "outer",
          smoothEdges: true,
          outputWidth: 370, // デフォルト横幅
          outputHeight: 320, // デフォルト縦幅
        });

        // セル（枠）の管理
        const [cells, setCells] = useState([]);
        const [selectedCellId, setSelectedCellId] = useState(null);

        // Canvas操作用
        const canvasRef = useRef(null);
        const [dragState, setDragState] = useState(null);
        const [creatingRect, setCreatingRect] = useState(null);

        // HEXからRGBへ変換
        const hexToRgb = (hex) => {
          const result =
            /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
          return result
            ? {
                r: parseInt(result[1], 16),
                g: parseInt(result[2], 16),
                b: parseInt(result[3], 16),
              }
            : null;
        };

        const handleImageUpload = (e) => {
          const file = e.target.files[0];
          if (file) {
            const reader = new FileReader();
            reader.onload = (event) => {
              const img = new Image();
              img.onload = () => {
                setImage(img);
                setImageUrl(event.target.result);
                setCells([]);
                setSelectedCellId(null);
              };
              img.src = event.target.result;
            };
            reader.readAsDataURL(file);
          }
        };

        // 単一セルの画像処理・プレビュー生成
        const processCellImage = useCallback(
          (cellGeometry, currentSettings, sourceImage) => {
            if (!sourceImage) return null;

            const { x, y, width, height } = cellGeometry;
            const {
              enableTransparent,
              targetColor,
              tolerance,
              mode,
              smoothEdges,
              outputWidth,
              outputHeight,
            } = currentSettings;

            // 範囲外チェック
            if (width <= 0 || height <= 0) return null;
            const sx = Math.max(0, Math.floor(x));
            const sy = Math.max(0, Math.floor(y));
            const sw = Math.min(Math.floor(width), sourceImage.width - sx);
            const sh = Math.min(Math.floor(height), sourceImage.height - sy);
            if (sw <= 0 || sh <= 0) return null;

            // 1. まず原寸サイズでCanvasを作成して透過処理を行う
            const tempCanvas = document.createElement("canvas");
            tempCanvas.width = sw;
            tempCanvas.height = sh;
            const ctx = tempCanvas.getContext("2d");

            ctx.drawImage(sourceImage, sx, sy, sw, sh, 0, 0, sw, sh);

            // 透過処理
            if (enableTransparent) {
              const targetRgb = hexToRgb(targetColor);
              if (targetRgb) {
                const imageData = ctx.getImageData(0, 0, sw, sh);
                const data = imageData.data;
                const { r: tr, g: tg, b: tb } = targetRgb;
                const tol = tolerance * 2.55;
                const tolSq = tol * tol * 3;

                const isMatch = (r, g, b, a) => {
                  if (a === 0) return true;
                  const distSq =
                    Math.pow(r - tr, 2) +
                    Math.pow(g - tg, 2) +
                    Math.pow(b - tb, 2);
                  return distSq <= tolSq;
                };

                if (mode === "all") {
                  for (let p = 0; p < data.length; p += 4) {
                    if (
                      isMatch(
                        data[p],
                        data[p + 1],
                        data[p + 2],
                        data[p + 3]
                      )
                    ) {
                      data[p + 3] = 0;
                    } else if (smoothEdges) {
                      const r = data[p],
                        g = data[p + 1],
                        b = data[p + 2];
                      const distSq =
                        Math.pow(r - tr, 2) +
                        Math.pow(g - tg, 2) +
                        Math.pow(b - tb, 2);
                      if (distSq <= tolSq * 1.2) {
                        const ratio =
                          (distSq - tolSq) / (tolSq * 0.2 || 1); // 安全ガード
                        data[p + 3] = Math.min(data[p + 3], ratio * 255);
                      }
                    }
                  }
                } else {
                  // Outer (Flood Fill)
                  const visited = new Uint8Array(sw * sh);
                  const queue = [];

                  for (let cx = 0; cx < sw; cx++) {
                    queue.push({ x: cx, y: 0 });
                    queue.push({ x: cx, y: sh - 1 });
                  }
                  for (let cy = 1; cy < sh - 1; cy++) {
                    queue.push({ x: 0, y: cy });
                    queue.push({ x: sw - 1, y: cy });
                  }

                  while (queue.length > 0) {
                    const { x: cx, y: cy } = queue.pop();
                    if (cx < 0 || cx >= sw || cy < 0 || cy >= sh) continue;
                    const idx = cy * sw + cx;
                    if (visited[idx]) continue;

                    const pIdx = idx * 4;
                    if (
                      isMatch(
                        data[pIdx],
                        data[pIdx + 1],
                        data[pIdx + 2],
                        data[pIdx + 3]
                      )
                    ) {
                      visited[idx] = 1;
                      data[pIdx + 3] = 0;
                      if (cx > 0) queue.push({ x: cx - 1, y: cy });
                      if (cx < sw - 1) queue.push({ x: cx + 1, y: cy });
                      if (cy > 0) queue.push({ x: cx, y: cy - 1 });
                      if (cy < sh - 1) queue.push({ x: cx, y: cy + 1 });
                    }
                  }
                }
                ctx.putImageData(imageData, 0, 0);
              }
            }

            // 2. 指定サイズに出力用Canvasを作成し、アスペクト比維持で描画
            const outW = outputWidth || 370;
            const outH = outputHeight || 320;

            const finalCanvas = document.createElement("canvas");
            finalCanvas.width = outW;
            finalCanvas.height = outH;
            const finalCtx = finalCanvas.getContext("2d");

            // リサイズ品質設定
            finalCtx.imageSmoothingEnabled = true;
            finalCtx.imageSmoothingQuality = "high";

            // 中央配置の計算 (Contain)
            const scale = Math.min(outW / sw, outH / sh);
            const drawW = sw * scale;
            const drawH = sh * scale;
            const drawX = (outW - drawW) / 2;
            const drawY = (outH - drawH) / 2;

            finalCtx.drawImage(
              tempCanvas,
              0,
              0,
              sw,
              sh,
              drawX,
              drawY,
              drawW,
              drawH
            );

            return finalCanvas.toDataURL("image/png");
          },
          []
        );

        // 設定変更時の再処理
        useEffect(() => {
          if (!image || cells.length === 0) return;
          setCells((prevCells) =>
            prevCells.map((cell) => ({
              ...cell,
              dataUrl: processCellImage(cell, settings, image),
            }))
          );
        }, [
          settings.enableTransparent,
          settings.targetColor,
          settings.tolerance,
          settings.mode,
          settings.smoothEdges,
          settings.outputWidth,
          settings.outputHeight,
        ]);

        // グリッド自動生成
        const initializeGrid = () => {
          if (!image) return;
          const { cols, rows } = settings;
          const cellWidth = Math.floor(image.width / cols);
          const cellHeight = Math.floor(image.height / rows);
          const newCells = [];

          for (let r = 0; r < rows; r++) {
            for (let c = 0; c < cols; c++) {
              const geo = {
                x: c * cellWidth,
                y: r * cellHeight,
                width: cellWidth,
                height: cellHeight,
              };
              newCells.push({
                id: Date.now() + r * cols + c,
                ...geo,
                name: `stamp_${String(newCells.length + 1).padStart(
                  2,
                  "0"
                )}.png`,
                dataUrl: processCellImage(geo, settings, image),
              });
            }
          }
          setCells(newCells);
          setSelectedCellId(null);
        };

        // --- Canvas Mouse Events ---

        const getMousePos = (e) => {
          const canvas = canvasRef.current;
          if (!canvas) return { x: 0, y: 0 };
          const rect = canvas.getBoundingClientRect();
          const scaleX = canvas.width / rect.width;
          const scaleY = canvas.height / rect.height;
          return {
            x: (e.clientX - rect.left) * scaleX,
            y: (e.clientY - rect.top) * scaleY,
          };
        };

        const getHitInfo = (x, y) => {
          const HANDLE_SIZE = 15;
          const list = selectedCellId
            ? [
                cells.find((c) => c.id === selectedCellId),
                ...cells.filter((c) => c.id !== selectedCellId),
              ]
            : [...cells].reverse();

          for (let cell of list) {
            if (!cell) continue;
            if (cell.id === selectedCellId) {
              if (
                Math.abs(x - (cell.x + cell.width)) < HANDLE_SIZE &&
                Math.abs(y - (cell.y + cell.height)) < HANDLE_SIZE
              ) {
                return { type: "resize", cell };
              }
            }
            if (
              x >= cell.x &&
              x <= cell.x + cell.width &&
              y >= cell.y &&
              y <= cell.y + cell.height
            ) {
              return { type: "move", cell };
            }
          }
          return null;
        };

        const handleMouseDown = (e) => {
          if (!image) return;
          const { x, y } = getMousePos(e);
          if (e.button !== 0) return;
          const hit = getHitInfo(x, y);

          if (hit) {
            setSelectedCellId(hit.cell.id);
            setDragState({
              type: hit.type,
              cellId: hit.cell.id,
              startX: x,
              startY: y,
              initialCell: { ...hit.cell },
            });
          } else {
            setSelectedCellId(null);
            setDragState({
              type: "create",
              startX: x,
              startY: y,
            });
            setCreatingRect({ x, y, width: 0, height: 0 });
          }
        };

        const handleMouseMove = (e) => {
          const { x, y } = getMousePos(e);
          if (!dragState) {
            const hit = getHitInfo(x, y);
            if (hit)
              document.body.style.cursor =
                hit.type === "resize" ? "nwse-resize" : "move";
            else document.body.style.cursor = "crosshair";
            return;
          }

          if (dragState.type === "create") {
            const w = x - dragState.startX;
            const h = y - dragState.startY;
            setCreatingRect({
              x: w < 0 ? x : dragState.startX,
              y: h < 0 ? y : dragState.startY,
              width: Math.abs(w),
              height: Math.abs(h),
            });
          } else {
            const dx = x - dragState.startX;
            const dy = y - dragState.startY;
            setCells((prev) =>
              prev.map((cell) => {
                if (cell.id !== dragState.cellId) return cell;
                if (dragState.type === "move") {
                  return {
                    ...cell,
                    x: dragState.initialCell.x + dx,
                    y: dragState.initialCell.y + dy,
                  };
                } else {
                  return {
                    ...cell,
                    width: Math.max(10, dragState.initialCell.width + dx),
                    height: Math.max(10, dragState.initialCell.height + dy),
                  };
                }
              })
            );
          }
        };

        const handleMouseUp = () => {
          if (!dragState) return;
          if (dragState.type === "create") {
            if (
              creatingRect &&
              creatingRect.width > 5 &&
              creatingRect.height > 5
            ) {
              const newId = Date.now();
              const newCell = {
                id: newId,
                ...creatingRect,
                name: `stamp_${String(cells.length + 1).padStart(
                  2,
                  "0"
                )}.png`,
              };
              newCell.dataUrl = processCellImage(newCell, settings, image);
              setCells((prev) => [...prev, newCell]);
              setSelectedCellId(newId);
            }
            setCreatingRect(null);
          } else {
            setCells((prev) =>
              prev.map((cell) => {
                if (cell.id === dragState.cellId) {
                  return {
                    ...cell,
                    dataUrl: processCellImage(cell, settings, image),
                  };
                }
                return cell;
              })
            );
          }
          setDragState(null);
          document.body.style.cursor = "default";
        };

        const removeSelectedCell = () => {
          if (selectedCellId) {
            setCells((prev) => prev.filter((c) => c.id !== selectedCellId));
            setSelectedCellId(null);
          }
        };

        const removeAllCells = () => {
          if (window.confirm("すべての枠を削除しますか？")) {
            setCells([]);
            setSelectedCellId(null);
          }
        };

        // 描画ループ
        useEffect(() => {
          const canvas = canvasRef.current;
          if (!canvas || !image) return;
          const ctx = canvas.getContext("2d");
          canvas.width = image.width;
          canvas.height = image.height;

          ctx.drawImage(image, 0, 0);
          ctx.fillStyle = "rgba(0, 0, 0, 0.4)";
          ctx.fillRect(0, 0, canvas.width, canvas.height);

          cells.forEach((cell, index) => {
            ctx.save();
            ctx.beginPath();
            ctx.rect(cell.x, cell.y, cell.width, cell.height);
            ctx.clip();
            ctx.drawImage(image, 0, 0);
            ctx.restore();

            const isSelected = cell.id === selectedCellId;
            ctx.strokeStyle = isSelected
              ? "#00ff00"
              : "rgba(255, 255, 255, 0.8)";
            ctx.lineWidth = isSelected ? 3 : 1;
            ctx.setLineDash(isSelected ? [] : [4, 4]);
            ctx.strokeRect(cell.x, cell.y, cell.width, cell.height);
            ctx.setLineDash([]);

            ctx.fillStyle = isSelected ? "#00ff00" : "rgba(0,0,0,0.5)";
            ctx.fillRect(cell.x, cell.y - 20, 30, 20);
            ctx.fillStyle = "#fff";
            ctx.font = "12px sans-serif";
            ctx.fillText(String(index + 1), cell.x + 8, cell.y - 6);

            if (isSelected) {
              ctx.fillStyle = "#00ff00";
              const hs = 10;
              ctx.fillRect(
                cell.x + cell.width - hs / 2,
                cell.y + cell.height - hs / 2,
                hs,
                hs
              );
            }
          });

          if (creatingRect) {
            ctx.strokeStyle = "#ffff00";
            ctx.lineWidth = 2;
            ctx.setLineDash([5, 5]);
            ctx.strokeRect(
              creatingRect.x,
              creatingRect.y,
              creatingRect.width,
              creatingRect.height
            );
            ctx.setLineDash([]);
            ctx.fillStyle = "rgba(255, 255, 255, 0.2)";
            ctx.fillRect(
              creatingRect.x,
              creatingRect.y,
              creatingRect.width,
              creatingRect.height
            );
          }
        }, [image, cells, selectedCellId, creatingRect]);

        const downloadAll = () => {
          if (!window.JSZip || cells.length === 0) return;
          const zip = new window.JSZip();
          cells.forEach((cell, idx) => {
            const fileName = `stamp_${String(idx + 1).padStart(2, "0")}.png`;
            if (cell.dataUrl) {
              const base64Data = cell.dataUrl.split(",")[1];
              zip.file(fileName, base64Data, { base64: true });
            }
          });
          zip.generateAsync({ type: "blob" }).then((content) => {
            const link = document.createElement("a");
            link.href = URL.createObjectURL(content);
            link.download = "stamps.zip";
            link.click();
          });
        };

        return (
          <div className="min-h-screen bg-gray-50 text-gray-800 font-sans p-4 md:p-8">
            <LoadJSZip onLoad={() => setJsZipLoaded(true)} />

            <div className="max-w-screen-2xl mx-auto">
              <header className="flex flex-col md:flex-row md:items-end justify-between mb-6 gap-4">
                <div>
                  <h1 className="text-2xl font-bold text-gray-900 flex items-center gap-2">
                    <ImageIcon className="w-8 h-8 text-blue-600" />
                    画像分割＆透過ツール
                  </h1>
                  <p className="text-sm text-gray-500 mt-1">
                    ドラッグで選択した範囲を、指定したサイズ（現在:{" "}
                    <span className="font-bold">
                      {settings.outputWidth}x{settings.outputHeight}px
                    </span>
                    ）で自動保存します。
                  </p>
                </div>
                <div className="flex gap-3">
                  {cells.length > 0 && (
                    <button
                      onClick={downloadAll}
                      className="bg-green-600 hover:bg-green-700 text-white px-6 py-2 rounded-lg font-bold shadow-sm flex items-center gap-2 transition-all active:scale-95"
                    >
                      <Save className="w-5 h-5" />
                      まとめて保存 ({cells.length}個)
                    </button>
                  )}
                </div>
              </header>

              <div className="grid grid-cols-1 lg:grid-cols-12 gap-6 items-start">
                <div className="lg:col-span-8 space-y-4">
                  <div className="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden flex flex-col">
                    {/* Toolbar Area */}
                    <div className="p-3 border-b border-gray-100 flex flex-col gap-2 bg-gray-50">
                      <div className="flex flex-wrap gap-2 justify-between items-center w-full">
                        <div className="flex items-center gap-4">
                          <span className="text-xs font-bold text-gray-500 uppercase tracking-wider flex items-center gap-1">
                            <MousePointer2 className="w-3 h-3" />
                            操作エリア
                          </span>

                          {image && (
                            <div className="flex gap-2">
                              <button
                                onClick={initializeGrid}
                                className="text-xs bg-white border border-gray-300 hover:bg-gray-100 px-3 py-1.5 rounded flex items-center gap-1 shadow-sm transition-colors"
                              >
                                <Grid className="w-3 h-3" />
                                自動グリッド配置
                              </button>
                              <button
                                onClick={removeAllCells}
                                disabled={cells.length === 0}
                                className="text-xs bg-white border border-gray-300 hover:bg-red-50 hover:text-red-600 hover:border-red-200 disabled:opacity-50 px-3 py-1.5 rounded flex items-center gap-1 shadow-sm transition-colors"
                              >
                                <Trash2 className="w-3 h-3" />
                                全削除
                              </button>
                            </div>
                          )}
                        </div>

                        {selectedCellId && (
                          <div className="flex items-center gap-2 bg-blue-50 px-3 py-1 rounded-full border border-blue-100 animate-fadeIn">
                            <span className="text-xs text-blue-800 font-medium">
                              選択中の枠を操作:
                            </span>
                            <button
                              onClick={removeSelectedCell}
                              className="text-xs text-red-600 hover:underline flex items-center gap-1"
                            >
                              <Trash2 className="w-3 h-3" /> 削除
                            </button>
                          </div>
                        )}
                      </div>

                      {image && (
                        <div className="text-[11px] text-gray-500 bg-white px-3 py-2 rounded border border-gray-200 flex flex-wrap gap-4 items-center">
                          <div className="flex items-center gap-1.5">
                            <span className="w-4 h-4 bg-gray-100 rounded flex items-center justify-center border text-[9px] font-bold">
                              1
                            </span>
                            <span>
                              画像上をドラッグして
                              <strong className="text-gray-700">
                                範囲を追加
                              </strong>
                            </span>
                          </div>
                          <div className="flex items-center gap-1.5 border-l pl-4">
                            <span className="w-4 h-4 bg-gray-100 rounded flex items-center justify-center border text-[9px] font-bold">
                              2
                            </span>
                            <span>
                              枠内クリックで
                              <strong className="text-gray-700">
                                選択・移動
                              </strong>
                            </span>
                          </div>
                          <div className="flex items-center gap-1.5 border-l pl-4">
                            <Maximize2 className="w-3 h-3 text-gray-400" />
                            <span>
                              枠の右下ドラッグで
                              <strong className="text-gray-700">
                                サイズ変更
                              </strong>
                            </span>
                          </div>
                        </div>
                      )}
                    </div>

                    <div className="relative bg-gray-500/10 flex items-center justify-center p-4 overflow-auto min-h-[500px] select-none">
                      {image ? (
                        <div className="relative shadow-xl inline-block">
                          <canvas
                            ref={canvasRef}
                            onMouseDown={handleMouseDown}
                            onMouseMove={handleMouseMove}
                            onMouseUp={handleMouseUp}
                            onMouseLeave={handleMouseUp}
                            className="max-w-full h-auto block bg-white cursor-crosshair"
                            style={{ maxHeight: "75vh", maxWidth: "100%" }}
                          />
                        </div>
                      ) : (
                        <div className="text-gray-400︓flex flex-col items-center py-24">
                          <Upload className="w-16 h-16 mb-4 opacity-20" />
                          <p className="text-lg font-medium">
                            画像をここにドロップ
                          </p>
                          <label className="bg-blue-600 hover:bg-blue-700 text-white px-6 py-2 rounded-lg cursor-pointer shadow-sm transition-transform active:scale-95 mt-4">
                            画像を選択
                            <input
                              type="file"
                              accept="image/png, image/jpeg"
                              onChange={handleImageUpload}
                              className="hidden"
                            />
                          </label>
                        </div>
                      )}
                    </div>
                  </div>
                </div>

                <div className="lg:col-span-4 space-y-4 h-full flex flex-col">
                  <div className="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden flex-shrink-0">
                    <div className="p-3 border-b border-gray-100 bg-gray-50/50 flex justify-between items-center">
                      <h2 className="font-bold text-gray-700 text-sm flex items-center gap-2">
                        <Settings className="w-4 h-4" />
                        共通設定
                      </h2>
                      <span className="text-[10px] text-gray-400">
                        変更は全スタンプに即時反映
                      </span>
                    </div>

                    <div className="p-4 space-y-4">
                      <div className="space-y-4">
                        <div className="bg-blue-50/50 p-3 rounded border border-blue-100 space-y-4">
                          {/* 出力サイズ設定 (縦・横) */}
                          <div>
                            <label className="text-xs font-bold text-gray-600 block mb-2 flex items-center gap-1">
                              <Maximize2 className="w-3 h-3" />
                              出力画像サイズ (px)
                            </label>
                            <div className="flex items-center gap-2">
                              <div className="flex-1">
                                <label className="text-[10px] text-gray-500 block mb-1">
                                  横幅 (W)
                                </label>
                                <input
                                  type="number"
                                  value={settings.outputWidth}
                                  onChange={(e) =>
                                    setSettings({
                                      ...settings,
                                      outputWidth: Math.max(
                                        1,
                                        Number(e.target.value)
                                      ),
                                    })
                                  }
                                  className="w-full border border-gray-300 rounded px-2 py-1.5 text-sm bg-white focus:ring-2 focus:ring-blue-500 outline-none"
                                />
                              </div>
                              <div className="text-gray-400 pt-5">×</div>
                              <div className="flex-1">
                                <label className="text-[10px] text-gray-500 block mb-1">
                                  縦幅 (H)
                                </label>
                                <input
                                  type="number"
                                  value={settings.outputHeight}
                                  onChange={(e) =>
                                    setSettings({
                                      ...settings,
                                      outputHeight: Math.max(
                                        1,
                                        Number(e.target.value)
                                      ),
                                    })
                                  }
                                  className="w-full border border-gray-300 rounded px-2 py-1.5 text-sm bg-white focus:ring-2 focus:ring-blue-500 outline-none"
                                />
                              </div>
                            </div>

                            {/* 推奨サイズリスト */}
                            <div className="mt-3">
                              <p className="text-[10px] text-gray-400 mb-2 flex items-center gap-1">
                                <Info className="w-3 h-3" /> クリックで推奨サイズを適用
                              </p>
                              <div className="space-y-1">
                                {PRESET_SIZES.map((preset, idx) => (
                                  <button
                                    key={idx}
                                    onClick={() =>
                                      setSettings({
                                        ...settings,
                                        outputWidth: preset.w,
                                        outputHeight: preset.h,
                                      })
                                    }
                                    className="w-full flex items-center justify-between text-[11px] p-2 bg-white border border-gray-200 rounded hover:bg-blue-50 hover:border-blue-300 transition-colors text-left group"
                                  >
                                    <div>
                                      <div className="font-bold text-gray-700">
                                        {preset.label}
                                      </div>
                                      <div className="text-[9px] text-gray-400">
                                        {preset.desc}
                                      </div>
                                    </div>
                                    <div className="text-right">
                                      <div className="font-mono text-blue-600 font-medium group-hover:text-blue-700">
                                        {preset.w}{" "}
                                        <span className="text-gray-400 text-[9px]">
                                          x
                                        </span>{" "}
                                        {preset.h}
                                      </div>
                                      <div className="text-[9px] text-gray-300 group-hover:text-blue-400">
                                        px
                                      </div>
                                    </div>
                                  </button>
                                ))}
                              </div>
                            </div>
                          </div>

                          <hr className="border-blue-100/50" />

                          <label className="flex items-center cursor-pointer gap-2">
                            <input
                              type="checkbox"
                              checked={settings.enableTransparent}
                              onChange={(e) =>
                                setSettings({
                                  ...settings,
                                  enableTransparent: e.target.checked,
                                })
                              }
                              className="w-4 h-4 text-blue-600 rounded"
                            />
                            <span className="text-sm font-medium text-gray-700">
                              背景透過を有効にする
                            </span>
                          </label>

                          {settings.enableTransparent && (
                            <>
                              <div className="flex items-center gap-2">
                                <div className="w-8 h-8 rounded-full border shadow-sm overflow-hidden flex-shrink-0 relative">
                                  <input
                                    type="color"
                                    value={settings.targetColor}
                                    onChange={(e) =>
                                      setSettings({
                                        ...settings,
                                        targetColor: e.target.value,
                                      })
                                    }
                                    className="absolute inset-0 w-full h-full p-0 border-0 opacity-0 cursor-pointer"
                                  />
                                  <div
                                    className="w-full h-full"
                                    style={{
                                      backgroundColor: settings.targetColor,
                                    }}
                                  />
                                </div>
                                <div className="flex-1">
                                  <p className="text-xs text-gray-500">
                                    透過色を選択
                                  </p>
                                  <p className="text-xs font-mono font-bold text-gray-700">
                                    {settings.targetColor}
                                  </p>
                                </div>
                              </div>

                              <div className="space-y-1">
                                <p className="text-xs text-gray-500">
                                  透過モード
                                </p>
                                <div
                                  className="flex rounded-md shadow-sm"
                                  role="group"
                                >
                                  <button
                                    onClick={() =>
                                      setSettings({
                                        ...settings,
                                        mode: "outer",
                                      })
                                    }
                                    className={`flex-1 text-xs py-1.5 px-2 border rounded-l-md transition-colors ${
                                      settings.mode === "outer"
                                        ? "bg-blue-600 text-white border-blue-600"
                                        : "bg-white text-gray-600 border-gray-300 hover:bg-gray-50"
                                    }`}
                                  >
                                    外側のみ
                                  </button>
                                  <button
                                    onClick={() =>
                                      setSettings({
                                        ...settings,
                                        mode: "all",
                                      })
                                    }
                                    className={`flex-1 text-xs py-1.5 px-2 border-t border-b border-r rounded-r-md transition-colors ${
                                      settings.mode === "all"
                                        ? "bg-blue-600 text-white border-blue-600"
                                        : "bg-white text-gray-600 border-gray-300 hover:bg-gray-50"
                                    }`}
                                  >
                                    全体
                                  </button>
                                </div>
                              </div>

                              <div>
                                <div className="flex justify-between text-xs text-gray-500 mb-1">
                                  <span>
                                    許容誤差（消えすぎたら下げてください）
                                  </span>
                                  <span className="font-bold">
                                    {settings.tolerance}
                                  </span>
                                </div>
                                <input
                                  type="range"
                                  min="0"
                                  max="100"
                                  value={settings.tolerance}
                                  onChange={(e) =>
                                    setSettings({
                                      ...settings,
                                      tolerance: Number(e.target.value),
                                    })
                                  }
                                  className="w-full h-1.5 bg-gray-200 rounded-lg appearance-none cursor-pointer accent-blue-600"
                                />
                              </div>
                            </>
                          )}
                        </div>
                      </div>
                    </div>
                  </div>

                  <div className="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden flex-1 flex flex-col min-h-[300px]">
                    <div className="p-3 border-b border-gray-100 flex justify-between items-center bg-gray-50">
                      <h3 className="font-bold text-gray-700 text-sm flex items-center gap-2">
                        <Layers className="w-4 h-4" />
                        生成プレビュー ({cells.length})
                      </h3>
                    </div>

                    <div className="flex-1 overflow-y-auto p-3 bg-gray-100">
                      {cells.length > 0 ? (
                        <div className="space-y-2">
                          {cells.map((cell, index) => (
                            <div
                              key={cell.id}
                              className={`bg-white p-2 rounded-lg border shadow-sm flex items-center gap-3 transition-colors cursor-pointer ${
                                cell.id === selectedCellId
                                  ? "ring-2 ring-blue-500 border-blue-500"
                                  : "border-gray-200 hover:border-blue-300"
                              }`}
                              onClick={() => setSelectedCellId(cell.id)}
                            >
                              <div className="w-6 h-6 flex items-center justify-center bg-gray-100 rounded text-xs text-gray-500 font-bold">
                                {index + 1}
                              </div>

                              <div className="w-16 h-16 bg-[url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAMUlEQVQ4T2N89uzZfwY8QFJSEp80A+OoAcMiDP7//483HTx//hx/Ohg1gIFx6ICDAwACV9WCXwdbRwAAAABJRU5ErkJggg==')] rounded border border-gray-100 flex-shrink-0 flex items-center justify-center">
                                {cell.dataUrl ? (
                                  <img
                                    src={cell.dataUrl}
                                    alt=""
                                    className="max-w-full max-h-full object-contain"
                                  />
                                ) : (
                                  <div className="animate-pulse bg-gray-200 w-full h-full" />
                                )}
                              </div>

                              <div className="flex-1 min-w-0">
                                <p className="text-xs font-medium text-gray-700 truncate">
                                  stamp_{String(index + 1).padStart(2, "0")}.png
                                </p>
                                <p className="text-[10px] text-gray-400">
                                  {settings.outputWidth} x{" "}
                                  {settings.outputHeight} px
                                </p>
                              </div>

                              <button
                                onClick={(e) => {
                                  e.stopPropagation();
                                  setCells((prev) =>
                                    prev.filter((c) => c.id !== cell.id)
                                  );
                                  if (selectedCellId === cell.id)
                                    setSelectedCellId(null);
                                }}
                                className="text-gray-400 hover:text-red-500 p-1 rounded hover:bg-red-50"
                                title="削除"
                              >
                                <X className="w-4 h-4" />
                              </button>
                            </div>
                          ))}
                        </div>
                      ) : (
                        <div className="h-full flex flex-col items-center justify-center text-gray-400 p-4 text-center">
                          <Plus className="w-8 h-8 mb-2 opacity-20" />
                          <p className="text-sm font-medium">
                            リストは空です
                          </p>
                          <p className="text-xs mt-1">
                            左の画像上をドラッグして
                            <br />
                            切り抜く範囲を指定してください
                          </p>
                        </div>
                      )}
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        );
      };

      const root = ReactDOM.createRoot(document.getElementById("root"));
      root.render(<ImageSplitter />);
    </script>
  </body>
</html>
