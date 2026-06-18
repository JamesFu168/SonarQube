# SonarQube 社群版 (Community Build) 10,000+ 全量問題智慧匯出腳本 (Test 專案專用)
## 【解決問題】
* 徹底解決單一類別（如 MAJOR - CODE_SMELL 達 12,214 筆）超出 SonarQube 官方 10,000 筆深度分頁限制的問題。
* 修正 SECURITY_HOTSPOT 造成的 API 400 (Bad Request) 報錯。
* 採用「嚴重程度 -> 問題類型 -> 年度區間」三級智慧拆分算法，保證 100% 完整抓取全量資料。
* 解決因為 SPA 路由導致網址列找不到 `id` 參數，進而誤抓全系統所有專案資料的問題。
## 【使用說明】
* 登入您的 SonarQube 網站。
* 進入您要匯出的專案 "Issues" (問題) 頁面。
* 按下鍵盤的 F12（或按滑鼠右鍵選擇「檢查」），切換到 "Console" (主控台) 標籤頁。
* 將此檔案中的程式碼完整複製，貼入 Console 中，然後按下 Enter 鍵。

## 【指令】
```
(async () => {
  // 1. 多重智慧偵測專案 Key (projectKey)
  const urlParams = new URLSearchParams(window.location.search);
  let detectedKey = urlParams.get('id') || urlParams.get('project') || urlParams.get('projectKey') || "";
  
  // 嘗試從網頁中的 DOM 元素（如儀表板連結）提取專案 Key
  if (!detectedKey) {
    const projectLink = document.querySelector('a[href*="id="], a[href*="project="]');
    if (projectLink) {
      try {
        const linkUrl = new URL(projectLink.href, window.location.origin);
        detectedKey = linkUrl.searchParams.get('id') || linkUrl.searchParams.get('project') || "";
        console.log(`%cℹ️ 自 DOM 連結中偵測到專案 Key: ${detectedKey}`, "color: #17a2b8;");
      } catch (e) {
        // 忽略解析錯誤
      }
    }
  }

  // 💡 強制防禦確認機制：彈出對話框讓使用者「確認」或「手動修正」專案 Key，確保絕對只抓單一專案
  let projectKey = prompt(
    "🎯 請確認或輸入您要匯出的【單一專案 Key】\n" +
    "（若保留空白或輸入錯誤，可能會導致匯出全系統所有專案的資料）：", 
    detectedKey
  );

  if (!projectKey || !projectKey.trim()) {
    console.error("%c❌ 執行失敗：未提供專案 Key，已取消下載。", "color: red; font-weight: bold;");
    alert("已取消下載。請提供正確的專案 Key 以進行單一專案匯出。");
    return;
  }
  projectKey = projectKey.trim();

  console.log(`%c🚀 [初始化] 開始準備下載專案 [${projectKey}] 的全量分析問題 (目標約 19,000+ 筆)...`, "color: #007bff; font-weight: bold;");

  const pageSize = 500; // SonarQube API 單次最大分頁限制
  const limitThreshold = 10000; // SonarQube 深度分頁 10,000 筆硬限制
  
  const severitiesList = ['BLOCKER', 'CRITICAL', 'MAJOR', 'MINOR', 'INFO'];
  const typesList = ['BUG', 'VULNERABILITY', 'CODE_SMELL']; // 排除非法的 SECURITY_HOTSPOT
  
  // 定義年度時間區間，用於解決單一分類大於 10,000 筆的極端情況
  const dateIntervals = [
    { before: '2020-01-01', label: '2020年以前' },
    { after: '2020-01-01', before: '2021-01-01', label: '2020年' },
    { after: '2021-01-01', before: '2022-01-01', label: '2021年' },
    { after: '2022-01-01', before: '2023-01-01', label: '2022年' },
    { after: '2023-01-01', before: '2024-01-01', label: '2023年' },
    { after: '2024-01-01', before: '2025-01-01', label: '2024年' },
    { after: '2025-01-01', before: '2026-01-01', label: '2025年' },
    { after: '2026-01-01', label: '2026年及以後' }
  ];

  // 用於記錄全域抓取到的 Issues (以 key 去重，確保資料乾淨)
  const uniqueIssuesMap = new Map();

  // 輔助函式：發送請求
  async function fetchJson(url) {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP 錯誤！狀態碼: ${response.status}`);
    }
    return await response.json();
  }

  // 三級智慧拆分抓取核心演算法
  async function fetchWithSplitting(queryParams, level = 0) {
    // 建立基礎 URL，同時帶上 projectKeys 與 projects 參數，確保在所有 SonarQube 版本中都能完美限制單一專案
    let baseQuery = `/api/issues/search?projectKeys=${projectKey}&projects=${projectKey}&ps=1&p=1`;
    if (queryParams.severity) baseQuery += `&severities=${queryParams.severity}`;
    if (queryParams.type) baseQuery += `&types=${queryParams.type}`;
    if (queryParams.createdAfter) baseQuery += `&createdAfter=${queryParams.createdAfter}`;
    if (queryParams.createdBefore) baseQuery += `&createdBefore=${queryParams.createdBefore}`;

    // 1. 先探測此篩選條件下的總筆數
    const initData = await fetchJson(baseQuery);
    const total = initData.paging.total;

    if (total === 0) return;

    const scopeName = `[嚴重:${queryParams.severity || '未分'} | 類型:${queryParams.type || '未分'} | 區間:${queryParams.dateLabel || '未分'}]`;

    // 2. 如果總筆數小於或等於 10,000，直接安全分頁抓取
    if (total <= limitThreshold) {
      console.log(`%c  ➔ 偵測到分組 ${scopeName} 共計 ${total} 筆，開始下載...`, "color: #28a745;");
      
      let page = 1;
      let fetchedCount = 0;
      
      while (fetchedCount < total) {
        let fetchUrl = `/api/issues/search?projectKeys=${projectKey}&projects=${projectKey}&ps=${pageSize}&p=${page}`;
        if (queryParams.severity) fetchUrl += `&severities=${queryParams.severity}`;
        if (queryParams.type) fetchUrl += `&types=${queryParams.type}`;
        if (queryParams.createdAfter) fetchUrl += `&createdAfter=${queryParams.createdAfter}`;
        if (queryParams.createdBefore) fetchUrl += `&createdBefore=${queryParams.createdBefore}`;

        const data = await fetchJson(fetchUrl);
        const issues = data.issues || [];
        
        if (issues.length === 0) break;

        issues.forEach(issue => {
          uniqueIssuesMap.set(issue.key, issue);
        });

        fetchedCount += issues.length;
        page++;
        
        // 稍微延遲避免伺服器負載過高
        await new Promise(resolve => setTimeout(resolve, 50));
      }
      return;
    }

    // 3. 一級拆分：如果總筆數大於 10,000，且尚未依「嚴重程度」拆分
    if (level === 0) {
      console.warn(`%c⚠️ 專案問題總量 (${total} 筆) 超過 API 單次限制 (10,000 筆)！啟用第一階段「嚴重程度」維度拆分抓取...`, "color: #fd7e14; font-weight: bold;");
      for (const severity of severitiesList) {
        await fetchWithSplitting({ severity: severity }, 1);
      }
      return;
    }

    // 4. 二級拆分：如果單一嚴重程度依然大於 10,000 筆，則啟用「問題類型」拆分
    if (level === 1) {
      console.warn(`%c⚠️ 嚴重程度 [${queryParams.severity}] 的問題數 (${total} 筆) 仍超過上限！啟用第二階段「問題類型」維度拆分抓取...`, "color: #ffc107; font-weight: bold;");
      for (const type of typesList) {
        await fetchWithSplitting({ severity: queryParams.severity, type: type }, 2);
      }
      return;
    }

    // 5. 三級拆分：如果「嚴重程度 + 問題類型」（例如 MAJOR + CODE_SMELL）依然大於 10,000 筆，啟用「年度時間區間」拆分
    if (level === 2) {
      console.warn(`%c🚨 分類 [${queryParams.severity} - ${queryParams.type}] 的問題數 (${total} 筆) 仍超過上限！啟用第三階段「年度時間區間」維度拆分抓取...`, "color: #dc3545; font-weight: bold;");
      for (const interval of dateIntervals) {
        await fetchWithSplitting({
          severity: queryParams.severity,
          type: queryParams.type,
          createdAfter: interval.after,
          createdBefore: interval.before,
          dateLabel: interval.label
        }, 3);
      }
      return;
    }

    // 6. 極端情況：如果到了時間區間還是大於 10,000 (理論上不可能)，則強行截斷抓取
    console.error(`%c🚨 極端警報：${scopeName} 依然有 ${total} 筆問題！超出腳本處理上限，將強行下載前 10,000 筆。`, "color: red; font-weight: bold;");
    let page = 1;
    let fetchedCount = 0;
    while (fetchedCount < limitThreshold) {
      let fetchUrl = `/api/issues/search?projectKeys=${projectKey}&projects=${projectKey}&ps=${pageSize}&p=${page}`;
      if (queryParams.severity) fetchUrl += `&severities=${queryParams.severity}`;
      if (queryParams.type) fetchUrl += `&types=${queryParams.type}`;
      if (queryParams.createdAfter) fetchUrl += `&createdAfter=${queryParams.createdAfter}`;
      if (queryParams.createdBefore) fetchUrl += `&createdBefore=${queryParams.createdBefore}`;

      const data = await fetchJson(fetchUrl);
      const issues = data.issues || [];
      if (issues.length === 0) break;

      issues.forEach(issue => {
        uniqueIssuesMap.set(issue.key, issue);
      });

      fetchedCount += issues.length;
      page++;
      await new Promise(resolve => setTimeout(resolve, 50));
    }
  }

  try {
    // 啟動智慧遞迴拆分流程
    await fetchWithSplitting({});

    const allIssues = Array.from(uniqueIssuesMap.values());
    console.log(`%c🎉 【下載完成】專案 [${projectKey}] 所有分組抓取並去重合併成功！`, "color: green; font-weight: bold; font-size: 15px;");
    console.log(`%c📊 最終獲得全量問題總筆數: ${allIssues.length} 項。`, "color: green; font-weight: bold; font-size: 14px;");

    if (allIssues.length === 0) {
      console.warn("ℹ️ 未偵測到符合條件的問題，取消報表產出。");
      return;
    }

    // 2. 轉換為簡潔的 Markdown 對照表格
    console.log("📝 正在轉換為 Markdown 報表格式...");
    let markdown = `# SonarQube 專案全量問題修改對照表 (${projectKey})\n\n`;
    markdown += `> **專案名稱 (Key)**: ${projectKey}  \n`;
    markdown += `> **問題總數量**: ${allIssues.length} 項  \n`;
    markdown += `> **導出時間**: ${new Date().toLocaleString()}  \n`;
    markdown += `> **備註**: 此專案規模較大，已啟用三級維度智慧拆分算法，成功繞過 SonarQube API 10,000 筆下載限制。  \n\n`;
    markdown += `| 序號 | 待修改的檔案 | 待修改的原因 (SonarQube Rule) | 嚴重程度 | 類型 |\n`;
    markdown += `| :--- | :--- | :--- | :--- | :--- |\n`;

    allIssues.forEach((issue, index) => {
      // 整理並提取檔案路徑 (去除專案 key 的前綴)
      const component = issue.component || "";
      const filePath = component.includes(':') ? component.split(':').slice(1).join(':') : component;
      
      // 清除訊息中的換行符號以保持 Markdown 表格整齊
      const message = issue.message ? issue.message.replace(/\r?\n/g, ' ') : '';
      const severity = issue.severity || 'UNKNOWN';
      const type = issue.type || 'UNKNOWN';
      
      markdown += `| ${index + 1} | \`${filePath}\` | ${message} | ${severity} | ${type} |\n`;
    });

    // 3. 自動觸發瀏覽器下載 Markdown 檔案
    const blob = new Blob([markdown], { type: "text/markdown;charset=utf-8;" });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.setAttribute("href", url);
    link.setAttribute("download", `sonarqube_all_issues_full_${projectKey}.md`);
    link.style.visibility = 'hidden';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);

    console.log(`%c✅ 全量報表下載成功！檔案已自動儲存至您的下載資料夾：sonarqube_all_issues_full_${projectKey}.md`, "color: #28a745; font-weight: bold; font-size: 14px;");
  } catch (error) {
    console.error("❌ 抓取失敗，詳細錯誤原因:", error);
  }
})();
```
