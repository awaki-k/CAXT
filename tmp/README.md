```
export type Priority = "low" | "medium" | "high";

export type Task = {
    id: string;
    text: string;
    done: boolean;
    collapsed?: boolean;
    children: Task[];
    priority?: Priority;
    due?: string;   // ISO-8601 date string (e.g., "2025-08-14")
    note?: string;
    createdAt: Date;
    completedAt?: Date; // Property to record the completion date and time
};
```eof

---
### 2. `src/components/DailyReportModal.tsx` （新規作成）

次に、日報作成の一連の流れを担当する、新しい専用のモーダルコンポーネントを作成します。

```react:DailyReportModal.tsx:src/components/DailyReportModal.tsx
import React, { useState } from 'react';
import type { Task } from '../types';

// The structure of the analysis result passed from App.tsx
export type ReportAnalysis = {
  completedTasks: Task[];
  problemTasks: Task[]; // Overdue or upcoming tasks
};

interface DailyReportModalProps {
  analysis: ReportAnalysis;
  onClose: () => void;
  onGenerate: (userResponses: Record<string, { blocker: string; progress: string }>) => Promise<void>;
  isGenerating: boolean;
  generatedReport: string | null;
}

export const DailyReportModal: React.FC<DailyReportModalProps> = ({
  analysis,
  onClose,
  onGenerate,
  isGenerating,
  generatedReport,
}) => {
  // State to manage user's answers to questions
  const [responses, setResponses] = useState<Record<string, { blocker: string; progress: string }>>({});

  const handleResponseChange = (taskId: string, field: 'blocker' | 'progress', value: string) => {
    setResponses(prev => ({
      ...prev,
      [taskId]: {
        ...prev[taskId],
        [field]: value,
      },
    }));
  };

  const handleDownload = () => {
    if (!generatedReport) return;
    const blob = new Blob([generatedReport], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    const today = new Date().toISOString().slice(0, 10);
    a.download = `daily-report-${today}.txt`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  };
  
  const handleCopy = () => {
    if (!generatedReport) return;
    // Using the clipboard API
    navigator.clipboard.writeText(generatedReport).then(() => {
        // You can add a success message here if you like
    }).catch(err => {
      console.error('Failed to copy text: ', err);
    });
  };


  const renderForm = () => (
    <>
      <div className="modal-header">
        <h3>日報作成アシスタント</h3>
        <button className="icon-btn danger" onClick={onClose}>✕</button>
      </div>
      <div className="modal-body">
        <h4>✅ 最近完了したタスク</h4>
        {analysis.completedTasks.length > 0 ? (
          <ul>{analysis.completedTasks.map(t => <li key={t.id}>{t.text}</li>)}</ul>
        ) : (
          <p className="muted">過去24時間以内に完了したタスクはありません。</p>
        )}

        <h4>⚠️ 確認が必要なタスク</h4>
        {analysis.problemTasks.length > 0 ? (
          analysis.problemTasks.map(task => (
            <div key={task.id} className="form-row" style={{ marginBottom: '20px' }}>
              <label className="label" style={{ fontSize: '1em' }}>{task.text}</label>
              <textarea
                className="input textarea"
                rows={2}
                placeholder="どこで詰まっていますか？（任意）"
                value={responses[task.id]?.blocker || ''}
                onChange={(e) => handleResponseChange(task.id, 'blocker', e.target.value)}
              />
              <textarea
                className="input textarea"
                rows={2}
                placeholder="現在の進捗はどうですか？（任意）"
                value={responses[task.id]?.progress || ''}
                onChange={(e) => handleResponseChange(task.id, 'progress', e.target.value)}
              />
            </div>
          ))
        ) : (
          <p className="muted">遅延・期限間近のタスクはありません。</p>
        )}
        <div className="form-actions" style={{ marginTop: '20px' }}>
          <button className="btn primary" onClick={() => onGenerate(responses)}>AIで日報を生成</button>
        </div>
      </div>
    </>
  );

  const renderLoading = () => (
    <div className="modal-body" style={{ textAlign: 'center', padding: '40px' }}>
      <p>AIが日報を生成中です...</p>
      {/* Simple spinner */}
      <div className="spinner"></div>
    </div>
  );

  const renderResult = () => (
     <>
      <div className="modal-header">
        <h3>生成された日報</h3>
         <button className="icon-btn danger" onClick={onClose}>✕</button>
      </div>
      <div className="modal-body">
        <textarea
            className="input textarea"
            rows={15}
            value={generatedReport || ''}
            readOnly
        />
        <div className="form-actions" style={{ marginTop: '20px' }}>
          <button className="btn ghost" onClick={handleCopy}>コピー</button>
          <button className="btn ghost" onClick={handleDownload}>ダウンロード (.txt)</button>
          <button className="btn primary" onClick={onClose}>終了</button>
        </div>
      </div>
    </>
  );

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content glass" onClick={(e) => e.stopPropagation()}>
        {isGenerating ? renderLoading() : generatedReport ? renderResult() : renderForm()}
      </div>
    </div>
  );
};
```eof

---
### 3. `src/App.tsx` （変更）

司令塔として、日報モーダルの状態管理、データ分析、AIへのリクエストなど、中心的な役割を担います。

```typescript:App.tsx:src/App.tsx
import { useMemo, useState } from "react";
import InputPanel from "./components/InputPanel";
import TaskTree from "./components/TaskTree";
import SortTags, { SortKey, SortOrder } from "./components/SortTags";
import { DailyReportModal, ReportAnalysis } from './components/DailyReportModal';
import type { Task, Priority } from "./types";
import "./App.css";

type Filter = "all" | "active" | "done";
type Mode = "input" | "structure";

const priorityOrder: Record<Priority, number> = { high: 3, medium: 2, low: 1 };

export default function App() {
    const [tasks, setTasks] = useState<Task[]>(seed());
    const [filter, setFilter] = useState<Filter>("all");
    const [mode, setMode] = useState<Mode>("input");
    const [sortKey, setSortKey] = useState<SortKey>('createdAt');
    const [sortOrder, setSortOrder] = useState<SortOrder>('desc');

    const [noteModalTask, setNoteModalTask] = useState<Task | null>(null);

    // --- State for Daily Report Modal ---
    const [isReportModalOpen, setIsReportModalOpen] = useState(false);
    const [reportAnalysis, setReportAnalysis] = useState<ReportAnalysis | null>(null);
    const [isGenerating, setIsGenerating] = useState(false);
    const [generatedReport, setGeneratedReport] = useState<string | null>(null);
    // ------------------------------------

    const counts = useMemo(() => {
        const total = countTasks(tasks);
        const done = countTasks(tasks, (t) => t.done);
        return { total, done, active: total - done };
    }, [tasks]);

    const create = (p: { text: string; priority?: Priority; due?: string; note?: string }) => {
        const id =
            typeof crypto !== "undefined" && "randomUUID" in crypto
                ? crypto.randomUUID()
                : `${Date.now()}-${Math.random().toString(16).slice(2)}`;
        setTasks((prev) => [...prev, { id, text: p.text, done: false, children: [], priority: p.priority, due: p.due, note: p.note, createdAt: new Date() }]);
        setMode("structure");
    };

    const sortedTasks = useMemo(() => {
        const sort = (tasksToSort: Task[]): Task[] => {
            const sorted = [...tasksToSort].sort((a, b) => {
                let comparison = 0;
                switch (sortKey) {
                    case 'dueDate':
                        const dateA = a.due ? new Date(a.due).getTime() : Infinity;
                        const dateB = b.due ? new Date(b.due).getTime() : Infinity;
                        comparison = dateA - dateB;
                        break;
                    case 'priority':
                        const priorityA = a.priority ? priorityOrder[a.priority] : 0;
                        const priorityB = b.priority ? priorityOrder[b.priority] : 0;
                        comparison = priorityB - priorityA;
                        break;
                    case 'createdAt':
                        comparison = b.createdAt.getTime() - a.createdAt.getTime();
                        break;
                }
                return sortOrder === 'asc' ? comparison : -comparison;
            });
            return sorted.map(task => ({ ...task, children: sort(task.children) }));
        };
        return sort(tasks);
    }, [tasks, sortKey, sortOrder]);

    const toggleDone = (id: string) => {
        setTasks((prev) =>
          mapTasks(prev, (t) => {
            if (t.id === id) {
              const isNowDone = !t.done;
              return { ...t, done: isNowDone, completedAt: isNowDone ? new Date() : undefined };
            }
            return t;
          })
        );
    };

    const toggleCollapse = (id: string) => setTasks((prev) => mapTasks(prev, (t) => (t.id === id ? { ...t, collapsed: !t.collapsed } : t)));
    const removeTask = (id: string) => setTasks((prev) => removeById(prev, id).rest);
    const dropAsChild = (parentId: string, draggedId: string) => {
        setTasks((prev) => {
            if (isDescendant(prev, draggedId, parentId)) return prev;
            const { removed, rest } = removeById(prev, draggedId);
            if (!removed) return prev;
            return insertChild(rest, parentId, removed);
        });
    };
    const dropToRoot = (draggedId: string) => {
        setTasks((prev) => {
            const { removed, rest } = removeById(prev, draggedId);
            if (!removed) return prev;
            return [...rest, removed];
        });
    };

    const openNoteModal = (task: Task) => setNoteModalTask(task);
    const closeNoteModal = () => setNoteModalTask(null);

    // --- Daily Report Logic ---
    const handleOpenReportModal = () => {
        const allTasks = flattenTasks(tasks);
        const now = new Date();
        const twentyFourHoursAgo = new Date(now.getTime() - 24 * 60 * 60 * 1000);
        const threeDaysFromNow = new Date(now.getTime() + 3 * 24 * 60 * 60 * 1000);

        const completedTasks = allTasks.filter(t => t.done && t.completedAt && t.completedAt > twentyFourHoursAgo);
        const problemTasks = allTasks.filter(t => !t.done && t.due && (new Date(t.due) < now || new Date(t.due) < threeDaysFromNow));

        setReportAnalysis({ completedTasks, problemTasks });
        setGeneratedReport(null);
        setIsGenerating(false);
        setIsReportModalOpen(true);
    };

    const handleCloseReportModal = () => {
        setIsReportModalOpen(false);
    };

    const handleGenerateReport = async (userResponses: Record<string, { blocker: string; progress: string }>) => {
        if (!reportAnalysis) return;
        setIsGenerating(true);

        // 1. Format the data for the AI prompt
        let prompt = "# 今日のタスク進捗\n\n";
        prompt += "## ✅ 完了したタスク\n";
        if (reportAnalysis.completedTasks.length > 0) {
            reportAnalysis.completedTasks.forEach(t => prompt += `- ${t.text}\n`);
        } else {
            prompt += "過去24時間で完了したタスクはありません。\n";
        }

        prompt += "\n## ⚠️ 確認が必要なタスク\n";
        if (reportAnalysis.problemTasks.length > 0) {
            reportAnalysis.problemTasks.forEach(t => {
                prompt += `### ${t.text}\n`;
                const response = userResponses[t.id];
                if (response?.blocker) prompt += `- 詰まっている点: ${response.blocker}\n`;
                if (response?.progress) prompt += `- 進捗: ${response.progress}\n`;
                if (!response?.blocker && !response?.progress) prompt += "- ユーザーからの入力はありませんでした。\n";
            });
        } else {
            prompt += "遅延・期限間近のタスクはありません。\n";
        }
        
        const systemInstruction = `あなたは優秀なプロジェクトアシスタントです。
以下の進捗情報とユーザーの自己申告を基に、ポジティブな振り返りと次のアクションを促すような、簡潔な日報をマークダウン形式で作成してください。
ユーザーからの入力がないタスクについても、客観的な事実（遅延している、期限が近いなど）に基づいて、優しく注意を促してください。

出力形式:
# 〇月〇日の日報

## 今日の成果
（完了タスクを基に、ポジティブな要約を生成）

## 現在の課題と次のアクション
（遅延タスクとユーザーの回答を基に、課題点を整理し、具体的な次のアクションを提案）

## 今週のフォーカス
（期限が迫っているタスクに言及し、注意を促す）
`;

        // 2. Call the AI API
        try {
            // ここにあなたのGemini APIキーを入力してください
            const apiKey = "YOUR_GEMINI_API_KEY"; 
            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=${apiKey}`;
            
            const payload = {
              contents: [{ parts: [{ text: prompt }] }],
              systemInstruction: { parts: [{ text: systemInstruction }] },
            };

            const response = await fetch(apiUrl, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            if (!response.ok) {
                const errorBody = await response.text();
                throw new Error(`API error: ${response.status} ${response.statusText} - ${errorBody}`);
            }

            const result = await response.json();
            const reportText = result.candidates[0].content.parts[0].text;
            setGeneratedReport(reportText);

        } catch (error) {
            console.error("AI report generation failed:", error);
            setGeneratedReport("エラー：日報の生成に失敗しました。APIキーが正しいか確認するか、時間をおいて再度お試しください。");
        } finally {
            setIsGenerating(false);
        }
    };
    // --------------------------

    return (
        <main className="wrap">
            <div className="hero">
                <div className="hero-inner">
                    <h1>構造化タスクメモリー</h1>
                    <div className="chips">
                        <span className="chip">全 {counts.total}</span>
                        <span className="chip">未完 {counts.active}</span>
                        <span className="chip">完了 {counts.done}</span>
                    </div>
                    <div className="mode-switch" role="tablist" aria-label="画面切替">
                        <button className={`tab lg ${mode === "input" ? "active" : ""}`} onClick={() => setMode("input")}>1. 入力</button>
                        <button className={`tab lg ${mode === "structure" ? "active" : ""}`} onClick={() => setMode("structure")}>2. 構造表示</button>
                    </div>
                </div>
            </div>

            <div className="container">
                {mode === "input" ? (
                    <InputPanel onCreate={create} />
                ) : (
                    <section className="panel">
                        <div className="toolbar">
                             <div className="filters" role="tablist" aria-label="フィルタ">
                                <button className={`tab ${filter === "all" ? "active" : ""}`} onClick={() => setFilter("all")}>すべて</button>
                                <button className={`tab ${filter === "active" ? "active" : ""}`} onClick={() => setFilter("active")}>未完</button>
                                <button className={`tab ${filter === "done" ? "active" : ""}`} onClick={() => setFilter("done")}>完了</button>
                            </div>
                            <button className="btn" onClick={handleOpenReportModal}>日報を作成</button>
                            <button className="btn ghost" onClick={() => setMode("input")}>＋ 追加する</button>
                        </div>

                        <div className="toolbar">
                            <SortTags
                                currentSortKey={sortKey}
                                currentSortOrder={sortOrder}
                                onSortChange={(key, order) => {
                                    setSortKey(key);
                                    setSortOrder(order);
                                }}
                            />
                        </div>

                        <TaskTree
                            tasks={sortedTasks}
                            filter={filter}
                            onToggleDone={toggleDone}
                            onRemove={removeTask}
                            onToggleCollapse={toggleCollapse}
                            onDropAsChild={dropAsChild}
                            onDropToRoot={dropToRoot}
                            onOpenModal={openNoteModal}
                        />
                    </section>
                )}
            </div>

            <nav className="bottom-nav">
                <button className={`bn-item ${mode === "input" ? "active" : ""}`} onClick={() => setMode("input")}>入力</button>
                <button className={`bn-item ${mode === "structure" ? "active" : ""}`} onClick={() => setMode("structure")}>タスク一覧</button>
            </nav>
            
            {noteModalTask && (
              <div className="modal-overlay" onClick={closeNoteModal}>
                    <div className="modal-content glass" onClick={(e) => e.stopPropagation()}>
                        <div className="modal-header">
                            <h3>{noteModalTask.text} のメモ</h3>
                            <button className="icon-btn danger" onClick={closeNoteModal}>✕</button>
                        </div>
                        <div className="modal-body">
                            <p>{noteModalTask.note}</p>
                        </div>
                    </div>
              </div>
            )}
            
            {isReportModalOpen && reportAnalysis && (
                <DailyReportModal 
                    analysis={reportAnalysis}
                    onClose={handleCloseReportModal}
                    onGenerate={handleGenerateReport}
                    isGenerating={isGenerating}
                    generatedReport={generatedReport}
                />
            )}
        </main>
    );
}

function mapTasks(arr: Task[], f: (t: Task) => Task): Task[] {
    return arr.map((t) => ({ ...f(t), children: mapTasks(t.children, f) }));
}
function countTasks(arr: Task[], pred: (t: Task) => boolean = () => true): number {
    let c = 0;
    const walk = (xs: Task[]) => xs.forEach((t) => { if (pred(t)) c++; if (t.children.length) walk(t.children); });
    walk(arr);
    return c;
}
function removeById(arr: Task[], id: string): { removed: Task | null; rest: Task[] } {
    const rest: Task[] = [];
    let removed: Task | null = null;
    for (const t of arr) {
        if (t.id === id) { removed = { ...t, children: t.children.map(clone) }; continue; }
        const sub = removeById(t.children, id);
        if (sub.removed) { removed = sub.removed; rest.push({ ...t, children: sub.rest }); }
        else rest.push(t);
    }
    return { removed, rest };
}
function insertChild(arr: Task[], parentId: string, child: Task): Task[] {
    return arr.map((t) => (t.id === parentId ? { ...t, children: [...t.children, child] } : { ...t, children: insertChild(t.children, parentId, child) }));
}
function isDescendant(arr: Task[], ancestorId: string, maybeDescendantId: string): boolean {
    const a = findById(arr, ancestorId); if (!a) return false; return !!findById(a.children, maybeDescendantId);
}
function findById(arr: Task[], id: string): Task | null {
    for (const t of arr) { if (t.id === id) return t; const f = findById(t.children, id); if (f) return f; }
    return null;
}
function clone(t: Task): Task { return { ...t, children: t.children.map(clone) }; }

function flattenTasks(tasks: Task[]): Task[] {
  let flat: Task[] = [];
  tasks.forEach(task => {
    flat.push(task);
    if (task.children.length > 0) {
      flat = flat.concat(flattenTasks(task.children));
    }
  });
  return flat;
}

function seed(): Task[] {
    const id = () => (typeof crypto !== "undefined" && "randomUUID" in crypto ? crypto.randomUUID() : `${Date.now()}-${Math.random().toString(16).slice(2)}`);
    const now = new Date();
    let i = 30;
    const createdAt = () => new Date(now.getTime() - (i-- * 1000 * 3600 * 24)); 

    return [
        {
            id: id(), text: "卒論", done: false, createdAt: createdAt(),
            children: [
                { id: id(), text: "論文調査", done: false, createdAt: createdAt(), children: [
                        { id: id(), text: "論文A", done: true, children: [], priority: "medium", due: "2025-05-31", createdAt: createdAt(), completedAt: new Date() },
                        { id: id(), text: "論文B", done: false, children: [], priority: "high", due: "2025-06-15", createdAt: createdAt(), note: "最重要論文。念入りに読む。" },
                        { id: id(), text: "論文C", done: false, children: [], priority: "low", due: "2025-07-01", createdAt: createdAt() },
                    ], priority: "high", due: "2025-12-16" },
                { id: id(), text: "実装", done: false, createdAt: createdAt(), children: [
                        { id: id(), text: "環境構築", done: true, children: [], priority: "medium", due: "2025-06-15", createdAt: createdAt(), completedAt: new Date(now.getTime() - 12 * 60 * 60 * 1000) },
                        { id: id(), text: "基本機能", done: false, children: [], priority: "high", due: "2025-09-30", createdAt: createdAt(), note: "ユーザー認証（JWT）\nタスクのCRUD操作\n親子関係の紐付け" },
                    ], priority: "medium" },
            ],
        },
    ];
}
```eof

---
### 4. `App.css` （変更）

最後に、`DailyReportModal`で使うローディングスピナーのスタイルを末尾に追加します。

```css:App.css:src/App.css
/* ... (既存のCSSはすべてそのまま) ... */

.pill.memo-icon {
  background-color: rgba(245, 158, 11, .15);
  border-color: rgba(245, 158, 11, .35);
  /* アイコンボタンに合わせるためpill固有のスタイルを微調整 */
  padding: 0;
  width: 30px;
  height: 30px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  width: 90%;
  max-width: 600px;
  padding: 20px;
  border-radius: 16px;
  backdrop-filter: blur(10px);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid var(--bd);
  padding-bottom: 12px;
  margin-bottom: 12px;
}

.modal-header h3 {
  margin: 0;
  font-size: 1.2em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.modal-body {
  max-height: 70vh;
  overflow-y: auto;
}
.modal-body p {
  margin: 0;
  white-space: pre-wrap;
  overflow-wrap: break-word;
  color: var(--muted);
}
.modal-body ul {
  padding-left: 20px;
}

/* ▼▼▼ ローディングスピナー用のスタイルを追加 ▼▼▼ */
.spinner {
  border: 4px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  border-top: 4px solid var(--brand);
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  margin: 20px auto;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
```eof
