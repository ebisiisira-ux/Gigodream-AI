# Gigodream-AI
An app that creates full blown movie in seconds
import React, { useEffect, useMemo, useRef, useState } from "react"; import { motion, AnimatePresence } from "framer-motion"; import { Play, Square, Film, Users, Mic, Image as ImageIcon, Cloud, Settings as SettingsIcon, Download, Wand2, Sparkles, Clock, Plus, Minus, Upload, Check, Type, LayoutGrid, } from "lucide-react";

/**

Gigodream AI — MVP (single-file React) — JavaScript


---

Primary use: Create a full-blown movie (≤ 1 hour) from script + character images + AI voiceovers.

This file provides a production-style skeleton with a working UI, a lightweight on-device preview renderer,

cloud-provider adapter interfaces, emotional analysis stubs, and a watermark end-card in exports.

✅ User-friendly interface

✅ Multi-cloud ready (adapters: Vertex, OpenAI, AWS Bedrock, Local Fallback)

✅ AI voice lab (voice samples → model placeholder)

✅ Script editor + auto-fill placeholders

✅ Character boxes (choose count, names, images) from a corner selector

✅ Timeline auto-fit to 1 hour (3600s)

✅ Emotion inference from plot → passed to TTS (stubbed)

✅ Competitive Ads Lab in main menu

✅ Watermark placed at the END of every exported video

✅ RedMi 10A friendly: Lite Mode, low-memory renderer, progressive steps

Notes:

This MVP demonstrates the full flow and UX with pluggable cloud calls (stubs). Replace stub functions


with real API calls (Vertex AI, OpenAI, ElevenLabs, AWS Polly/Bedrock, etc.).

Exported preview video is silent by default (for device safety). Hook real TTS audio into the Exporter


when you integrate providers. The end-card watermark is rendered in-video. */


// ----------------------------- Utility: Tiny store & helpers ----------------------------- const clamp = (n, lo, hi) => Math.max(lo, Math.min(hi, n)); const seconds = (n) => n; const ONE_HOUR = 60 * 60; // 3600 sec

const defaultWatermark = { text: "Gigodream AI", sub: "Rendered with love", };

const defaultCharacters = (count) => Array.from({ length: count }).map((_, i) => ({ id: i + 1, name: Char ${i + 1}, img: null }));

// Simple emotion keywords (extend as needed) const EMOTION_LEXICON = [ { key: "grief", emotion: "sad", weight: 3 }, { key: "death", emotion: "sad", weight: 2 }, { key: "cry", emotion: "sad", weight: 2 }, { key: "love", emotion: "happy", weight: 2 }, { key: "joy", emotion: "happy", weight: 2 }, { key: "win", emotion: "happy", weight: 1 }, { key: "anger", emotion: "angry", weight: 2 }, { key: "rage", emotion: "angry", weight: 3 }, { key: "fight", emotion: "angry", weight: 2 }, { key: "fear", emotion: "fear", weight: 2 }, { key: "terror", emotion: "fear", weight: 3 }, { key: "shock", emotion: "surprise", weight: 2 }, { key: "wow", emotion: "surprise", weight: 1 }, ];

function inferEmotion(text) { const t = (text || "").toLowerCase(); const score = { happy: 0, sad: 0, angry: 0, fear: 0, surprise: 0, neutral: 0 }; for (const { key, emotion, weight } of EMOTION_LEXICON) { if (t.includes(key)) score[emotion] += weight; } const top = Object.entries(score).sort((a, b) => b[1] - a[1])[0]; const emotion = top && top[1] > 0 ? top[0] : "neutral"; const intensity = clamp((top?.[1] || 0) / 4, 0, 1); return { emotion, intensity }; }

// Parse script → scenes (super simple splitter) function parseScenes(script) { const cleaned = (script || "").trim(); if (!cleaned) return []; // Split by double newlines or \nSCENE markers const parts = cleaned .replace(/\n\sSCENE\s/gi, "\n\nSCENE: ") .split(/\n\n+/) .map((p) => p.trim()) .filter(Boolean); return parts.map((p, i) => ({ id: i + 1, text: p })); }

// Auto-fill placeholders like [CHAR1], [CHAR2], {{Char 1}}, {{Char 2}}, etc. function autofillScriptPlaceholders(script, characters) { let out = script; characters.forEach((c, idx) => { const n = idx + 1; const name = c.name || Char ${n}; out = out .replaceAll(new RegExp(\\[CHAR${n}\\], "g"), name) .replaceAll(new RegExp(\\{\\{\\s*Char\\s*${n}\\s*\\}\\}, "gi"), name) .replaceAll(new RegExp(\\{\\{\\s*${name}\\s*\\}\\}, "g"), name); }); return out; }

// Compute per-scene duration, scaled to ≤ 1 hour function fitDurationsToHour(scenes) { if (!scenes.length) return []; // naive read time: 180 wpm → 3 wps const words = scenes.map((s) => (s.text.split(/\s+/).filter(Boolean).length || 1)); const baseDurations = words.map((w) => Math.max(4, Math.round(w / 3))); // at least 4s per scene const total = baseDurations.reduce((a, b) => a + b, 0); if (total <= ONE_HOUR) return baseDurations; const scale = ONE_HOUR / total; return baseDurations.map((d) => Math.max(3, Math.floor(d * scale))); }

// ----------------------------- Cloud Provider Adapters (stubs) ----------------------------- class ProviderAdapter { constructor(keys = {}) { this.keys = keys; } id() { return "base"; } name() { return "Base"; } async ttsCloneAndSynthesize({ text, emotion, voiceSamples = [] }) { // Implement in subclasses. Return { audioUrl } or { arrayBuffer } throw new Error("Not implemented"); } }

class LocalFallbackAdapter extends ProviderAdapter { id() { return "local"; } name() { return "Local Fallback"; } async ttsCloneAndSynthesize({ text, emotion }) { // Placeholder: use SpeechSynthesis for preview (cannot export easily) if (typeof window !== "undefined" && window.speechSynthesis) { const u = new SpeechSynthesisUtterance(text); u.rate = 1; u.pitch = 1 + (emotion?.intensity || 0) * 0.2; // light affect try { speechSynthesis.speak(u); } catch {} } return { audioUrl: null }; } }

class VertexAdapter extends ProviderAdapter { id() { return "vertex"; } name() { return "Google Vertex"; } async ttsCloneAndSynthesize({ text, emotion, voiceSamples = [] }) { // TODO: Implement Vertex AI (Text-to-Speech / AudioLM) with voice cloning where available // Use this.keys.vertexKey, this.keys.vertexProject, this.keys.vertexLocation return { audioUrl: null }; } }

class OpenAIAdapter extends ProviderAdapter { id() { return "openai"; } name() { return "OpenAI"; } async ttsCloneAndSynthesize({ text, emotion, voiceSamples = [] }) { // TODO: Implement OpenAI Audio (TTS/Voice Clone when available) return { audioUrl: null }; } }

class BedrockAdapter extends ProviderAdapter { id() { return "bedrock"; } name() { return "AWS Bedrock"; } async ttsCloneAndSynthesize({ text, emotion, voiceSamples = [] }) { // TODO: Implement via Polly or Bedrock providers (ElevenLabs, etc.) return { audioUrl: null }; } }

const PROVIDERS = [new LocalFallbackAdapter(), new VertexAdapter(), new OpenAIAdapter(), new BedrockAdapter()];

// ----------------------------- Exporter: Canvas → WebM (with end watermark) ----------------------------- async function renderStoryboardWebM({ canvas, ctx, width, height, scenes, durations, characters, watermark, fps = 30, lite = true, }) { // Safety for low-end devices: use MediaRecorder from canvas capture const stream = canvas.captureStream(fps); const rec = new MediaRecorder(stream, { mimeType: "video/webm;codecs=vp9" }); const chunks = []; rec.ondataavailable = (e) => e.data.size && chunks.push(e.data);

let resolveDone; const done = new Promise((res) => (resolveDone = res)); rec.onstop = () => resolveDone(); rec.start();

const drawCenteredText = (text, y, size = 28, maxWidth = width - 80) => { ctx.font = ${size}px ui-sans-serif, system-ui; ctx.fillStyle = "#fff"; ctx.textAlign = "center"; // wrap const words = text.split(/\s+/); let line = ""; const lines = []; const lh = size * 1.3; for (const w of words) { const tmp = line ? line + " " + w : w; if (ctx.measureText(tmp).width > maxWidth) { lines.push(line); line = w; } else { line = tmp; } } if (line) lines.push(line); let yy = y; lines.forEach((ln) => { ctx.fillText(ln, width / 2, yy); yy += lh; }); };

const drawCharacterRow = () => { const size = Math.min(96, Math.floor(width / Math.max(3, characters.length))); const pad = 12; const totalW = characters.length * (size + pad) - pad; let x = (width - totalW) / 2; const y = height - size - 24; characters.forEach((c) => { // frame ctx.fillStyle = "rgba(0,0,0,0.35)"; ctx.fillRect(x - 6, y - 6, size + 12, size + 12); if (c.__img) ctx.drawImage(c.__img, x, y, size, size); ctx.strokeStyle = "rgba(255,255,255,0.6)"; ctx.strokeRect(x, y, size, size); // name ctx.font = 12px ui-sans-serif; ctx.fillStyle = "#e5e7eb"; ctx.textAlign = "center"; ctx.fillText(c.name || "?", x + size / 2, y + size + 14); x += size + pad; }); };

// Preload character images into canvas Image for (const c of characters) { if (c.img) { const img = new Image(); img.src = c.img; await new Promise((res) => { img.onload = () => res(); img.onerror = () => res(); }); c.__img = img; } }

const bg = (ctx, i) => { const hue = (i * 47) % 360; const grad = ctx.createLinearGradient(0, 0, width, height); grad.addColorStop(0, hsl(${hue},60%,16%)); grad.addColorStop(1, hsl(${(hue + 60) % 360},65%,22%)); ctx.fillStyle = grad; ctx.fillRect(0, 0, width, height); };

// Draw all scenes let sceneIndex = 0; for (const scene of scenes) { const dur = durations[sceneIndex] || 4; // seconds const totalFrames = Math.max(1, Math.floor(dur * fps));

for (let f = 0; f < totalFrames; f++) {
  bg(ctx, sceneIndex);
  // Scene header
  ctx.font = "16px ui-sans-serif"; ctx.fillStyle = "#a3e635"; ctx.textAlign = "left";
  ctx.fillText(`Scene ${sceneIndex + 1}/${scenes.length}`, 24, 36);

  // Emotion glow bar (top-right)
  const { emotion, intensity } = inferEmotion(scene.text);
  const emo = emotion.toUpperCase();
  const glowW = 160; const glowH = 8;
  ctx.fillStyle = "rgba(0,0,0,0.4)"; ctx.fillRect(width - glowW - 24, 22, glowW, glowH);
  ctx.fillStyle = "#fafafa"; ctx.fillRect(width - glowW - 24, 22, Math.max(8, glowW * intensity), glowH);
  ctx.font = "10px ui-sans-serif"; ctx.fillStyle = "#e5e7eb"; ctx.textAlign = "right";
  ctx.fillText(emo, width - 24, 18);

  // Main text
  drawCenteredText(scene.text, height / 2 - 40, lite ? 22 : 28);

  // Character strip
  drawCharacterRow();

  await new Promise((res) => requestAnimationFrame(res));
}

sceneIndex++;

}

// End watermark card (always appended) for (let f = 0; f < fps * 3; f++) { // 3 seconds ctx.fillStyle = "#000"; ctx.fillRect(0, 0, width, height); ctx.font = "bold 36px ui-sans-serif"; ctx.fillStyle = "#fff"; ctx.textAlign = "center"; ctx.fillText(watermark.text || "Gigodream AI", width / 2, height / 2); ctx.font = "16px ui-sans-serif"; ctx.fillStyle = "#9ca3af"; ctx.fillText(watermark.sub || "", width / 2, height / 2 + 28); await new Promise((res) => requestAnimationFrame(res)); }

rec.stop(); await done; return new Blob(chunks, { type: "video/webm" }); }

// ----------------------------- UI Primitives (simple, tailwind classes) ----------------------------- function TabButton({ icon: Icon, label, active, onClick }) { return ( <button onClick={onClick} className={flex items-center gap-2 w-full px-3 py-2 rounded-xl transition ${ active ? "bg-emerald-500 text-white" : "bg-zinc-800 hover:bg-zinc-700 text-zinc-200" }} > <Icon size={16} /> <span className="text-sm font-medium">{label}</span> </button> ); }

function Card({ children }) { return <div className="rounded-2xl bg-zinc-900/70 border border-zinc-800 shadow-xl p-4">{children}</div>; }

function Field({ label, children, hint }) { return ( <div className="flex flex-col gap-2"> <div className="text-xs uppercase tracking-wide text-zinc-400">{label}</div> {children} {hint ? <div className="text-[10px] text-zinc-500">{hint}</div> : null} </div> ); }

// ----------------------------- Main Component ----------------------------- export default function GigodreamAI() { const [active, setActive] = useState("studio"); // studio | voice | script | ads | settings const [providerId, setProviderId] = useState("local"); const [keys, setKeys] = useState({ vertexKey: "", vertexProject: "", vertexLocation: "", openaiKey: "", awsKey: "", awsSecret: "" }); const [characterCount, setCharacterCount] = useState(3); const [characters, setCharacters] = useState(defaultCharacters(3)); const [script, setScript] = useState(""); const [liteMode, setLiteMode] = useState(true); // optimize for Redmi 10A const [watermark, setWatermark] = useState(defaultWatermark); const [voiceSamples, setVoiceSamples] = useState([]); // File objects const [rendering, setRendering] = useState(false); const [videoUrl, setVideoUrl] = useState(null);

// keep characters length in sync useEffect(() => { setCharacters((prev) => { const next = [...prev]; if (characterCount > prev.length) { for (let i = prev.length; i < characterCount; i++) next.push({ id: i + 1, name: Char ${i + 1}, img: null }); } else if (characterCount < prev.length) { next.length = characterCount; } return next; }); }, [characterCount]);

const parsedScenes = useMemo(() => parseScenes(autofillScriptPlaceholders(script, characters)), [script, characters]); const durations = useMemo(() => fitDurationsToHour(parsedScenes), [parsedScenes]); const totalSec = useMemo(() => durations.reduce((a, b) => a + b, 0), [durations]);

const provider = useMemo(() => { const found = PROVIDERS.find((p) => p.id() === providerId) || PROVIDERS[0]; found.keys = keys; return found; }, [providerId, keys]);

// Canvas ref for render const canvasRef = useRef(null); const ctxRef = useRef(null); useEffect(() => { if (canvasRef.current && !ctxRef.current) { ctxRef.current = canvasRef.current.getContext("2d"); } }, []);

const handleImageUpload = (i, file) => { const reader = new FileReader(); reader.onload = (e) => setCharacters((prev) => prev.map((c, idx) => (idx === i ? { ...c, img: e.target.result } : c))); reader.readAsDataURL(file); };

const addVoiceSamples = (files) => setVoiceSamples((prev) => [...prev, ...Array.from(files)]);

const synthPreviewLine = async (text) => { const em = inferEmotion(text); await provider.ttsCloneAndSynthesize({ text, emotion: em, voiceSamples }); };

const doRender = async () => { if (!canvasRef.current || !ctxRef.current) return; setRendering(true); setVideoUrl(null); const width = liteMode ? 640 : 1280; const height = liteMode ? 360 : 720; const canvas = canvasRef.current; canvas.width = width; canvas.height = height; try { const blob = await renderStoryboardWebM({ canvas, ctx: ctxRef.current, width, height, scenes: parsedScenes, durations, characters, watermark, fps: 30, lite: liteMode, }); const url = URL.createObjectURL(blob); setVideoUrl(url); } catch (e) { alert("Render failed: " + e.message); } finally { setRendering(false); } };

return ( <div className="min-h-screen bg-gradient-to-b from-[#0a0a0a] to-[#111] text-white"> {/* Header */} <div className="sticky top-0 z-10 backdrop-blur border-b border-zinc-800 bg-zinc-950/60"> <div className="max-w-6xl mx-auto px-4 py-3 flex items-center justify-between"> <div className="flex items-center gap-3"> <div className="w-8 h-8 rounded-xl bg-emerald-500 grid place-items-center font-black">G</div> <div> <div className="font-semibold">Gigodream AI</div> <div className="text-[11px] text-zinc-400">Movie Studio • Voice Lab • Ads Lab</div> </div> </div> <div className="flex items-center gap-3 text-xs"> <span className="hidden sm:inline text-zinc-400">Total Runtime</span> <div className="px-2 py-1 rounded-lg bg-zinc-800 flex items-center gap-1"><Clock size={14} /> {Math.floor(totalSec / 60)}m {totalSec % 60}s</div> <div className="px-2 py-1 rounded-lg bg-zinc-800 flex items-center gap-2"> <Cloud size={14} /> <select className="bg-transparent outline-none" value={providerId} onChange={(e) => setProviderId(e.target.value)}> {PROVIDERS.map((p) => ( <option key={p.id()} value={p.id()}>{p.name()}</option> ))} </select> </div> <button onClick={() => setActive("settings")} className="px-2 py-1 rounded-lg bg-zinc-800 hover:bg-zinc-700"><SettingsIcon size={16} /></button> </div> </div> </div>

{/* Body */}
  <div className="max-w-6xl mx-auto px-4 py-6 grid grid-cols-12 gap-4">
    {/* Sidebar */}
    <div className="col-span-12 lg:col-span-3 space-y-2">
      <TabButton icon={Film} label="Movie Studio" active={active === "studio"} onClick={() => setActive("studio")} />
      <TabButton icon={Mic} label="Voice Lab" active={active === "voice"} onClick={() => setActive("voice")} />
      <TabButton icon={Type} label="Script Editor" active={active === "script"} onClick={() => setActive("script")} />
      <TabButton icon={Sparkles} label="Ads Lab" active={active === "ads"} onClick={() => setActive("ads")} />
      <TabButton icon={SettingsIcon} label="Settings" active={active === "settings"} onClick={() => setActive("settings")} />

      <Card>
        <Field label="Lite Mode (Redmi 10A)">
          <label className="inline-flex items-center gap-2 text-sm">
            <input type="checkbox" checked={liteMode} onChange={(e) => setLiteMode(e.target.checked)} />
            Enable low-memory render & smaller preview
          </label>
        </Field>
      </Card>
    </div>

    {/* Main */}
    <div className="col-span-12 lg:col-span-9 space-y-4">
      {active === "studio" && (
        <Studio
          characters={characters}
          setCharacters={setCharacters}
          characterCount={characterCount}
          setCharacterCount={setCharacterCount}
          parsedScenes={parsedScenes}
          durations={durations}
          script={script}
          setActive={setActive}
          synthPreviewLine={synthPreviewLine}
          doRender={doRender}
          rendering={rendering}
          videoUrl={videoUrl}
          watermark={watermark}
          canvasRef={canvasRef}
        />
      )}
      {active === "voice" && (
        <VoiceLab voiceSamples={voiceSamples} addVoiceSamples={addVoiceSamples} synthPreviewLine={synthPreviewLine} />
      )}
      {active === "script" && (
        <ScriptEditor script={script} setScript={setScript} characters={characters} />
      )}
      {active === "ads" && <AdsLab />}
      {active === "settings" && (
        <Settings keys={keys} setKeys={setKeys} watermark={watermark} setWatermark={setWatermark} />
      )}
    </div>
  </div>
</div>

); }

// ----------------------------- Sections ----------------------------- function Studio({ characters, setCharacters, characterCount, setCharacterCount, parsedScenes, durations, script, setActive, synthPreviewLine, doRender, rendering, videoUrl, watermark, canvasRef }) { const totalSec = durations.reduce((a, b) => a + b, 0); return ( <div className="space-y-4"> <Card> <div className="flex items-center justify-between"> <div className="font-semibold text-lg">Storyboard & Characters</div> <div className="flex items-center gap-3 text-sm"> <div className="hidden sm:flex items-center gap-2 text-zinc-400"><LayoutGrid size={16}/> Boxes</div> <div className="flex items-center gap-2 bg-zinc-800 rounded-lg px-2 py-1"> <button onClick={() => setCharacterCount((c) => clamp(c - 1, 1, 12))} className="p-1 hover:bg-zinc-700 rounded"><Minus size={14}/></button> <div className="w-10 text-center">{characterCount}</div> <button onClick={() => setCharacterCount((c) => clamp(c + 1, 1, 12))} className="p-1 hover:bg-zinc-700 rounded"><Plus size={14}/></button> </div> </div> </div> <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-3 mt-3"> {characters.map((c, i) => ( <div key={c.id} className="rounded-2xl border border-zinc-800 bg-zinc-900/50 overflow-hidden"> <div className="relative aspect-square grid place-items-center bg-zinc-900"> {c.img ? ( <img src={c.img} alt={c.name} className="object-cover w-full h-full" /> ) : ( <div className="text-zinc-500 text-xs flex flex-col items-center"> <ImageIcon className="mb-1" size={18} /> Drop Image </div> )} <label className="absolute inset-0 cursor-pointer"> <input type="file" accept="image/*" className="hidden" onChange={(e) => e.target.files?.[0] && handleProxyUpload(i, e.target.files[0], setCharacters)} /> </label> <div className="absolute top-2 right-2 text-[10px] px-2 py-1 rounded-full bg-zinc-800/80">#{i + 1}</div> </div> <div className="p-2"> <input value={c.name} onChange={(e) => setCharacters((prev) => prev.map((cc, idx) => (idx === i ? { ...cc, name: e.target.value } : cc)))} className="w-full bg-zinc-800 rounded-lg px-2 py-1 text-sm outline-none" p
