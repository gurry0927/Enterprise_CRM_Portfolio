# Enterprise 專案管理系統 - 技術思維與實作細節

> 這份文件不只是一份專案紀錄，它也是我正在建構的「個人數位分身（Digital Twin）」的底層知識庫。
> 這裡面記載了我在極限 45 天內，獨自從零打造這套 Enterprise CRM 系統的架構決策、技術挑戰與突破，以及為了符合企業級規範所做的每一步權衡。

---

## 1. 架構演進的決策過程

### 1.1 從 GAS 到統一 Odoo 平台

**開發初期的技術債**（Enterprise 1.0 / GAS 時代）：
```
資料分散問題:
- Enterprise-CRS LINE Bot (GAS)     → Google Sheets
- Enterprise-CRS WebUI (GAS)         → Google Sheets
- Enterprise Odoo 模組 (enterprise_crm)   → PostgreSQL

結果: 三個獨立系統，同步複雜，沒有 Single Source of Truth
```

**GAS 平台的限制**：
- 6 分鐘執行 timeout
- 100KB cache 限制
- 10,000 筆資料效能瓶頸
- sharded index workaround 複雜且脆弱

**我向主管與團隊提出的架構重構提案**：
- 終止資料分裂，以 Odoo 作為唯一資料源（PostgreSQL 15）
- 統一開發語言，全面轉向 Python 3.10+
- 採用單一 VM、單一 docker-compose 集中部署
- 確立三層架構：Web App (React) → FastAPI → Odoo RPC

**當時為什麼選 Odoo 而不是純 Firestore？**
```
當時的思考：
✅ Odoo:
  - 已有 enterprise_crm 模組（聯絡人管理成熟）
  - PostgreSQL 提供強大查詢能力
  - ORM 提供 ACID 事務保證

❌ Firestore:
  - 無事務支持（複雜操作風險）
  - 後來才因 LINE Bot 需要即時訪問而引入
```

---

### 1.2 Firestore 遷移的技術轉折：權限整合與客製化彈性 (2026-02-23)

**背景與痛點**：當系統從「內部管理」延伸到「LINE Bot 現場應用」與「Web 高階統計」時，營運團隊與我同時發現原有的 Odoo 平台遭遇了管理複雜度與顯示彈性的雙重瓶頸。

**我主導的決策樹與架構移轉**：
```
問題：雙重認證摩擦（Auth Friction）與 UI 擴充受限

├─ 方案 A: 留在 Odoo 並嘗試修改 View
│  └─ 限制: Odoo 原生視圖語法較為僵化，難以達成主管要求的複雜客製化 UI 需求。
│
├─ 方案 B: 維持 Odoo + Firebase 雙系統認證
│  └─ 限制: 員工需管理兩套帳密，開發端維護兩套權限同步（RBAC）的複雜度太高。
│
└─ 方案 C: 核心資料遷移至 Firestore ✅ 我的最終提案
   └─ 優勢:
      - 統一使用 Firebase Auth (Google SSO)，大幅降低員工管理負擔。
      - 現有案量規模（約 2,000 筆）完美適配 Firestore，效能極佳且幾乎免維護。
      - 前端 React 可 100% 自定義 UI，完美滿足營運端對報表的客製化要求。
```

**遷移後的架構**：
```
┌─────────────────────────────────────────┐
│ Firestore (讀寫優化)                     │
│ ├─ contacts, companies, groups, projects│
│ ├─ 用於 Web App, LINE Bot 快速訪問      │
│ └─ Single source for API層              │
├─────────────────────────────────────────┤
│ Odoo (業務邏輯)                         │
│ ├─ 原有 enterprise_crm 模組保留               │
│ ├─ 內部工作流和複雜業務邏輯             │
│ └─ 後端仍可調用（如需）                 │
└─────────────────────────────────────────┘
```

**ID 映射策略**：
```python
# Odoo → Firestore 的 ID 轉換
Odoo ID 123 (enterprise.group)           → "grp_123"
Odoo ID 456 (res.partner/company)  → "com_456"
Odoo ID 789 (res.partner/contact)  → "cnt_789"
Odoo ID "CE25240" (project)        → "prj_CE25240"
```

---

## 2. 核心設計模式

### 2.1 「專案為關係中心」架構

這是我在架構中最重要的核心決策。

**傳統 CRM 的問題**：
```python
# 業界標準模型
Contact --parent_id--> Company --group_id--> Group
          ↑ 1:1 or 1:N（綁死關係）

# 問題：同一人只能屬於一家公司
# 但現實中：吳柏諺 可能同時代表 集鈞顧問、集登設計、展延電機
```

**我的設計**：
```python
# Enterprise 模型（project-centric）
Contact ◄────── ProjectContact ──────► Project
               ↓
         company_ids[] (可多選)

# 實際例子：同一個人，多個專案，多個公司
吳柏諺 在 CE25240_華陰街 代表：
  - 集鈞顧問（主要）
  - 集登設計（技術）
  - 展延電機（採購）

吳柏諺 在 SC9954_錦西街 代表：
  - 集鈞顧問（只代表一家）
```

**資料結構設計**：
```python
# 聯絡人的兩層關係
Contact層:  parent_id (optional) 或 group_id (optional)
           → 預設歸屬，可為空

Project層: project_contact.company_ids[]
          → 在此專案代表的公司（多選）
          → 完全取代或補充 Contact 層

# 查詢範例：找出聯絡人的所有關聯公司
def search_contact_companies(contact_id):
    """查詢聯絡人的所有關聯公司（聯集）"""
    # 層級 1: 直屬公司
    direct = contact.parent_id

    # 層級 2: 專案關聯公司
    from_projects = db.collection('project_contacts')\
        .where('contactId', '==', contact_id)\
        .stream()

    all_company_ids = set()
    for pc in from_projects:
        all_company_ids.update(pc.get('companyIds', []))

    if direct:
        all_company_ids.add(direct)

    return list(all_company_ids)
```

---

### 2.2 樂觀更新 (Optimistic UI) 提升操作體驗

**問題場景**：
在傳統的 CRUD 應用中，當使用者點擊「合併聯絡人」或「更新資料」時，前端必須等待後端 API (FastAPI) 與 Firestore 交互完成後，才能重新拉取資料並重繪畫面。在網路不穩或資料庫負載高時，這會造成長達 1~2 秒的畫面凍結 (Loading spinner)，體驗極差。

**我的設計與實作**：
為了打造流暢的企業級應用，我引入了 **樂觀更新 (Optimistic UI)** 模式。

```text
[使用者點擊更新] 
     │
     ├─ 1. 前端同步更新：假裝 API 已經成功，直接覆寫 React Store (Zustand/Context) 裡的狀態
     ├─ 2. 畫面瞬間重繪：使用者感覺「不到 0.1 秒就完成了」
     │
     └─ 3. 背景非同步發送 API 給後端
           ├─ 若 API 成功 (99% 情況)：不動聲色，完美銜接
           └─ 若 API 失敗 (1% 情況)：觸發 rollback 機制，將畫面恢復到更新前的狀態，並拋出 Toast 錯誤提示
```

**效益**：
對於高頻操作（如：連續勾選專案成員、快速編輯電話欄位）的場景，樂觀更新徹底消除了網路延遲帶來的摩擦感，讓網頁端應用 (Web App) 達到了接近原生 App 的絲滑體驗。

---

### 2.3 幽靈狀態與前端狀態同步 (State Synchronization)

**問題場景**：
在多表單操作時，經常發生所謂的「資料不一致」。例如：在「聯絡人詳細頁」刪除了該聯絡人，退回「專案列表頁」時，那個已經被刪除的聯絡人居然還顯示在畫面上，點進去卻出現 404 Error。傳統做法是強制整個頁面重新整理 (`window.location.reload()`)，但這違反了 SPA (Single Page Application) 的體驗。

**我的解法**：全局狀態攔截與修復

我設計了一套「自我修復 (Self-Healing)」的前端緩存機制：
```javascript
// 核心概念：在全局請求攔截器中捕獲 404
api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 404) {
      // 擷取不存在的 ID
      const missingId = extractIdFromUrl(error.config.url);
      
      // 直接通知全局 Store 清除這個幽靈 ID
      globalStore.getState().removeStaleItem(missingId);
      
      // 畫面根據 Store 自動響應式消失
      toast.warning('該筆資料已被刪除或不存在，畫面已自動同步');
    }
    throw error;
  }
);
```

**高階思考**：
與其在每一個 Component 裡面寫 `if (error.status === 404) { setList(...) }`，我將這個資料同步的邏輯提升到全局的 API Interceptor 層級。這樣一來，不論使用者是在哪一個頁面踩到已刪除的資料，系統都會自動把它從全局緩存中剔除，實現「被動式但絕對一致」的資料同步。

---

## 3. 複雜資料的語意化清洗與正規化 (Data Sanitization)

**設計哲學**：在企業系統中，資料的「一致性」決定了「可搜尋性」。我堅持 **Input-Side Sanitization (輸入側清洗)**：在資料進入資料庫的那一刻，就透過自研的規則引擎將混亂的字串轉化為具備語意標籤的結構化資料。

### 3.1 通訊欄位解析：市話、手機與分機的解構

**挑戰**：使用者常在同一個欄位輸入多組號碼，且格式各異（如：`02-23456789 #101 / 0912-345678`）。若不拆分，後端將無法進行精準的手機反查。

**關鍵設計決策**：
- **正規化為純數字**：移除所有括號、空格與減號，統一為搜尋引擎友善的格式。
- **國際碼平滑轉換**：自動識別 `+886` 並轉換為本地撥號格式 `09...`。
- **遞迴拆分邏輯**：使用正則表達式自動識別 `\n`, `/`, `,` 等分隔符號，實現「單一輸入、多重存儲」。

**實作範例**：
```python
def normalize_phone(phone: Optional[str]) -> Optional[str]:
    """
    正規化電話號碼為純數字格式，方便比對
    """
    if not phone:
        return None

    phone = phone.strip()

    # 無效值過濾
    if phone in ("無", "無電話", "N/A", "-", ""):
        return None

    # 移除所有非數字和加號
    cleaned = re.sub(r"[^\d+]", "", phone)

    # 國際碼轉換（關鍵！）
    if cleaned.startswith("+886"):
        # +886912345678 → 0912345678
        cleaned = "0" + cleaned[4:]
    elif cleaned.startswith("886"):
        # 886912345678 → 0912345678
        cleaned = "0" + cleaned[3:]

    # 最小長度驗證（台灣電話最短 8 位市話）
    if len(cleaned) < 8:
        return None

    return cleaned


def format_phone_display(phone: Optional[str]) -> Optional[str]:
    """
    將正規化的電話號碼格式化為可讀格式
    """
    phone = normalize_phone(phone)
    if not phone:
        return None

    # 市話格式化
    if phone.startswith("02"):
        return f"{phone[:2]}-{phone[2:6]}-{phone[6:]}"
    elif phone.startswith(("03", "04", "05", "06", "07", "08")):
        return f"{phone[:2]}-{phone[2:5]}-{phone[5:]}"

    # 手機格式化
    elif phone.startswith("09"):
        return f"{phone[:4]}-{phone[4:7]}-{phone[7:]}"

    return phone


def split_phone_and_mobile(value: str) -> dict:
    """
    分離混合欄位中的市話和手機

    輸入: "04-2213-8848 #851\n0918-774-526"
    輸出: {
        'phone': '04-2213-8848',
        'ext': '851',
        'mobile': '0918-774-526'
    }
    """
    result = {'phone': None, 'ext': None, 'mobile': None}

    # 分離多行/多號碼
    parts = re.split(r'[\n,/]', value)

    for part in parts:
        part = part.strip()

        # 分離分機
        if '#' in part:
            phone_part, ext_part = part.split('#', 1)
            part = phone_part.strip()
            result['ext'] = ext_part.strip()

        normalized = normalize_phone(part)
        if not normalized:
            continue

        # 判斷是市話還是手機
        if normalized.startswith('09'):
            if not result['mobile']:
                result['mobile'] = format_phone_display(normalized)
        else:
            if not result['phone']:
                result['phone'] = format_phone_display(normalized)

    return result
```

---

### 3.2 啟發式台灣地址解析 (Heuristic Address Parsing)

**挑戰**：台灣地址的郵遞區號位置極度不固定（開頭、縣市後、或根本沒有）。直接用切割法會導致資料遺失。

**關鍵設計決策**：
- **位置無關的提取**：僅從「開頭」或「縣市後」提取 3-5 碼數字，避開街道中的門牌號碼，大幅降低誤判率。
- **全形半形自動校正**：在解析前進行統一的數字正規化處理。

**實作範例**：
```python
TAIWAN_CITIES = [
    '台北市', '臺北市', '新北市', '桃園市', '台中市', '台南市', '高雄市',
    '基隆市', '新竹市', '嘉義市',
    '新竹縣', '苗栗縣', '彰化縣', '南投縣', '雲林縣', '嘉義縣',
    '屏東縣', '宜蘭縣', '花蓮縣', '台東縣', '澎湖縣', '金門縣', '連江縣',
]

_CITY_PATTERN = '|'.join(TAIWAN_CITIES)


def parse_taiwan_address(address: Optional[str]) -> dict:
    """
    解析台灣地址，提取 zip, city, street

    設計決策：
    - 只從「開頭」或「縣市後面」提取郵遞區號
    - 不從街道中間的數字提取（避免誤判街號）
    """
    if not address:
        return {'zip': None, 'city': None, 'street': None}

    # 正規化
    address = address.strip()
    address = _normalize_digits(address)  # 全形→半形
    address = address.replace('臺', '台')  # 統一用「台」

    # Step 1: 提取縣市
    city_match = re.search(_CITY_PATTERN, address)
    if not city_match:
        return {'zip': None, 'city': None, 'street': address.strip()}

    city = city_match.group()
    city_start = city_match.start()
    city_end = city_match.end()

    # Step 2: 提取郵遞區號
    zip_code = None

    # 案例 1: 郵遞區號在縣市之前 ("224 新北市...")
    if city_start >= 3:
        before_city = address[:city_start].strip()
        match = re.search(r'(\d{3,5})\s*$', before_city)
        if match:
            zip_code = match.group(1)
            street = address[city_end:].strip()
            return {'zip': zip_code, 'city': city, 'street': street}

    # 案例 2: 郵遞區號在縣市之後 ("新北市22061板橋區...")
    after_city = address[city_end:]
    match = re.search(r'^(\d{3,5})', after_city)
    if match:
        zip_code = match.group(1)
        street = after_city[len(zip_code):].strip()
        return {'zip': zip_code, 'city': city, 'street': street}

    # 案例 3: 無郵遞區號
    street = address[city_end:].strip()
    return {'zip': None, 'city': city, 'street': street}
```

---

### 3.3 複維度重複聯絡人偵測演算法

**挑戰**：單純比對「姓名」會造成誤刪（同名同姓），單純比對「電話」會因為重複建檔而失效。

**關鍵設計決策**：
- **姓名 + 公司名稱雙軌匹配**：只有兩者皆符合「正規化後的特徵」才視為潛在重複。
- **姓名權威化 (Name Canonicalization)**：在比對前自動移除「職稱（經理、建築師）」與「括號備註」，確保比對的是真正的「實體」。

**實作範例**：
```python
def find_duplicate_contacts(name: str, company_name: str = None) -> List[dict]:
    """
    重複比對策略：「公司+姓名」組合匹配

    設計決策：
    - 只有姓名相同 AND 公司相同才視為重複
    - 允許同名但不同公司的人（常見情況）
    - 公司比對包含：直屬公司 + 專案關聯公司
    """

    # 正規化姓名（移除空格、稱謂）
    normalized_name = normalize_name(name)

    # 搜尋同名聯絡人
    candidates = db.collection('contacts')\
        .where('normalizedName', '==', normalized_name)\
        .stream()

    results = []
    for doc in candidates:
        contact = doc.to_dict()
        contact_id = doc.id

        # 取得該聯絡人的所有關聯公司
        contact_companies = get_all_contact_companies(contact_id)

        # 公司比對
        if company_name:
            normalized_company = normalize_company_name(company_name)

            # 檢查是否有任一公司匹配
            for company in contact_companies:
                if normalize_company_name(company['name']) == normalized_company:
                    results.append({
                        'contact': contact,
                        'matched_company': company['name'],
                        'match_type': 'exact'
                    })
                    break
        else:
            # 無公司資訊時，直接加入候選（讓使用者判斷）
            results.append({
                'contact': contact,
                'matched_company': None,
                'match_type': 'name_only'
            })

    return results


def normalize_name(name: str) -> str:
    """
    正規化姓名

    處理：
    - 移除空格：「王 大明」→「王大明」
    - 移除稱謂：「王大明 經理」→「王大明」
    - 移除括號：「王大明(舊)」→「王大明」
    """
    if not name:
        return ''

    # 移除常見稱謂後綴
    TITLES = ['經理', '總經理', '董事長', '協理', '副總', '主任',
              '工程師', '設計師', '建築師', '技師', '先生', '小姐']

    result = name.strip()
    for title in TITLES:
        result = result.replace(title, '')

    # 移除括號內容
    result = re.sub(r'\([^)]*\)', '', result)
    result = re.sub(r'（[^）]*）', '', result)

    # 移除所有空格
    result = result.replace(' ', '').replace('\u3000', '')

    return result


def normalize_company_name(name: str) -> str:
    """
    正規化公司名稱

    處理：
    - 移除括號附註：「台積電(新竹)」→「台積電」
    - 移除空格
    - 統一全形半形
    """
    if not name:
        return ''

    result = name.strip()

    # 移除括號內容
    result = re.sub(r'\([^)]*\)', '', result)
    result = re.sub(r'（[^）]*）', '', result)

    # 移除空格
    result = result.replace(' ', '').replace('\u3000', '')

    return result
```

---

## 4. 問題解決案例

### 4.1 NoSQL 資料模型重構：提升通訊錄的檢索效能 (2026-02-26)

**挑戰**：在早期的 Schema 設計中，為了保持結構彈性，我將所有聯絡方式（市話、手機、傳真）放進同一個 `phones` 陣列中。但隨著資料量增加，業務端反映「透過手機號碼反查聯絡人」的搜尋速度變慢，且邏輯變得異常複雜。

**場景還原：查詢效能的瓶頸**：
```python
# 早期 Schema (巢狀陣列)
contacts: {
  name: "王大明",
  phones: [
    { type: "office", number: "02-1234-5678" },
    { type: "mobile", number: "0912-345-678" }
  ]
}

# 瓶頸：Firestore 對於過度嵌套的陣列物件 (Array of Objects) 
# 無法建立高效的複合索引 (Composite Index)
```

**我的解法：Schema 扁平化與語意化分離**：
為了解決這個效能與查詢限制，我決定對 Firestore 的資料結構進行「反正規化（Denormalization）與扁平化」，將原先的單一陣列，拆解為兩個帶有明確業務語意的一維陣列：

```python
# 升級後的 Schema (扁平化陣列)
contacts: {
  name: "王大明",
  phones: ["02-1234-5678"],    # 專門存放市話/代表號
  mobiles: ["0912-345-678"],   # 專門存放手機
}
```

**重構效益**：
1. **O(1) 的查詢語法**：後端尋找特定手機號碼的查詢，從原本必須使用耗能的 `array-contains` 搭配手動物件解構，變成了極度輕量的 `where('mobiles', 'array_contains', '0912-345-678')`。
2. **索引支援**：Firestore 可以針對扁平化的 `mobiles` 陣列原生建立自動索引，大幅降低了讀取成本（Read Operations），同時讓前端表單綁定資料時的邏輯變得極度單純。

---

### 4.2 從概念驗證(MVP)到企業級架構：專案聯絡人的結構重構 (2026-02-26)

**挑戰**：系統在早期開發階段，為了快速驗證「專案聯絡人管理」的商業價值，團隊先使用 Google Apps Script (GAS) 搭配 Google Sheets 建立了第一版 MVP 測試。當主管確認需求方向可行，但受限於 GAS 效能瓶頸，決定由我主導重構成為基於 Firestore 的正規企業級系統時，遭遇了巨大的資料結構斷層。

**場景還原：MVP 與 Production 的架構碰撞**：
```
Phase 1 (MVP 時期 - 團隊以 GAS 寫入 Google Sheets):
  - 為了讓主管最快看到價值，報表上只直接存入純文字：contactName = "王大明"
  - 試算表本身沒有關聯式資料庫的概念，因此聯絡人缺乏實體 ID（contactId = NULL）

Phase 2 (Production 時期 - 升級至 Firestore):
  - 導入了嚴謹的 NoSQL 關聯設計，必須有明確的關係鍵結 contactId = "cnt_123" 才能建立專案關係
```
這導致部分完全依賴早期 MVP 建檔的歷史專案，在新的強關聯架構下，無法透過 ID 查找到聯絡人，導致前端畫面無法渲染。

**我的解法：雙層 Fallback 取代硬報錯，加上自動化收編機制**：
在進行系統升級的雙軌過渡期，我設計了雙層讀取策略與資料重整機制，確保在使用者營運體驗上「平滑過渡、零報錯」：

```python
def get_project_contacts(project_id: str):
    """
    雙層 Fallback 策略：
    在正式把所有 MVP 時期的歷史名單清洗乾淨前，系統必須能無痛兼容兩種格式。
    """
    results = []
    # ... 省略取資料 ...
    
        if contact_id:
            # 優先層：新系統嚴謹格式，透過 ID 反查即時姓名
            contact_doc = db.collection('contacts').document(contact_id).get()
            if contact_doc.exists:
                contact_name = contact_doc.to_dict().get('name', contact_name)
        
        # 退路層：即使沒有 ID（無關聯 ID 的歷史資料），依然拿出純文字，保證畫面正常渲染
        results.append({
            'contact_id': contact_id,      # 可能為空，但前端依然可渲染
            'contact_name': contact_name,  # 保底至少有文字
        })
    return results
```

**最終收斂（資料一致性修復腳本）**：
為了把這些 MVP 時期留下的「無關聯 ID 歷史實體」納入新系統的嚴謹管理，我開發了一套一次性的「自動關聯與收編腳本」。它會透過姓名反查主檔通訊錄，若命中則自動補齊 `contactId` 建立實體關聯；這樣便巧妙地解除了早期快速開發留下來的技術債，顯著提升資料庫的結構一致性。

---

### 4.3 合併引擎與可溯源資料清洗 (Data Cleaning with Traceability) (2026-03-08)

**挑戰**：系統中累積了 3,000+ 筆聯絡人，由於歷史原因（多人各自輸入），存在大量重複資料。在建立「合併引擎」時，如果直接採用傳統的「覆蓋 (Overwrite)」清洗模式，一旦發生誤併，原始資料將永久遺失，造成嚴重的孤兒資料 (Orphan Data) 問題。

**我的解法：將清洗視為「建立映射 (Mapping)」而非「覆寫」**：
秉持著我過去整理的開源設計模式 **[Data Cleaning with Traceability](https://github.com/gurry0927/data-cleaning-with-traceability)**，我將這個理念落實在 CRM 的合併引擎中：

1. **不可變的原始資料 (Raw Data Is Immutable)**：被合併的歷史聯絡人（Raw Value）不會被 `DELETE` 刪除，而是透過給予 `mergedIntoId` 標記為軟刪除狀態。
2. **清洗是建立關係 (Cleaning Is Mapping)**：當多筆資料發生衝突時，系統會引導管理員進行欄位選擇，最終結果將指向一個「權威實體 (Normalized Entity)」。
3. **無損回溯 (Reversibility)**：搭配全量寫入 Firestore 的稽核系統，所有的合併決策與關聯轉移都被記錄下來，確保未來任何時刻都能透過腳本實現可靠的反向還原。這套機制成功清理了數百筆歷史重複資料，並且達到了上線後「0 筆資料永久遺失」的安全目標。

---



### 5.1 從 PoC 到自動化：為 Excel 匯入精靈裝上後端引擎

**背景與分工**：同事最初設計了一個體驗極佳的「4 步驟 Excel 匯入精靈 UI」雛型。我接手這個 PoC 後，我們達成了明確的分工：他負責前端引導的流暢度，而我負責打造**後端的深層清洗與智慧比對引擎**，解決最棘手的資料庫關聯與資料正規化問題。
**設計理由（前端視角）**：分步驟減低使用者認知負荷

```
Step 1: 上傳
  ├─ 拖曳或點擊選擇 Excel
  ├─ 智慧專案偵測（從檔名提取 CE25240）
  └─ 解析 Sheet

Step 2: 欄位對應
  ├─ 自動匹配（關鍵字：姓名、公司、電話...）
  ├─ 手動調整（下拉選單）
  └─ 分機欄位特殊處理

Step 3: 預覽
  ├─ 表格顯示準備匯入的資料
  ├─ 電話清洗展示
  ├─ 地址解析展示
  └─ 重複檢測警告

Step 4: 結果
  ├─ 匯入進度
  ├─ 成功/失敗統計
  └─ 跳轉到詳情頁
```

**關鍵設計決策**：

```text
【邏輯 1：分機欄位整併】
- 挑戰：業務端的 Excel 習慣把分機獨立一個欄位，但資料庫為了搜尋效率統一存為單一字串。
- 解法：在資料清洗階段，讀取 [電話] 與 [分機] 兩欄，若分機有值，自動組合為 "02-2345-6789 #101" 再寫入資料庫。

【邏輯 2：合併儲存格向下填充】
- 挑戰：Excel 常見「多個聯絡人共用一家公司」的合併儲存格，導致程式讀取時，只有第一列有公司名，後面都是空值。
- 解法：設計一個狀態記憶體。當讀取每一列時：
   > 如果「公司欄位」有值：記住這個名字。
   > 如果「公司欄位」為空：自動填入剛剛記住的名字。
   > (這個簡單的邏輯成功解決了 80% 報表匯入的漏資料問題)
```

---

### 5.2 API 客戶端設計

為了讓前端與後端的溝通保持穩定與安全，我規劃了一套統一的 API 攔截機制（API Interceptor），而不是讓每個畫面各自去打 API。

**請求與驗證流程設計**：

```text
[前端元件 (React)] 
      │ 呼叫 api.get('/contacts')
      ▼
[API 攔截器 (Axios Interceptor)]
      │ 1. 攔截外出請求
      │ 2. 從 LocalStorage 抓取 JWT Token
      │ 3. 自動塞入 Header (Authorization: Bearer <token>)
      ▼
[後端伺服器 (FastAPI)]
      │
      ▼回傳 Response
[全局錯誤處理器]
      │ - 若回傳 200 OK → 直接交給前端元件渲染
      │ - 若回傳 401 Unauthorized → Token 過期，直接強制跳轉回 /login 登入頁
      │ - 若回傳 404 Not Found → 啟動「自癒同步」，從全局 Store 移除該項目 (見 2.3 節)
      │ - 若回傳 500 Error → 彈出統一的錯誤提示 Toast
```

**設計效益**：
這樣的架構設計讓前端的業務邏輯變得非常乾淨，開發新的頁面時完全不需要管「Token 有沒有帶」、「過期要怎麼處理」，全部由底層的客戶端統一接管。

---

## 6. 技術棧總覽

```
層級              技術           用途
─────────────────────────────────────────────────────
前端層
  框架            React 19       SPA 應用
  UI 庫           Ant Design 6   企業級元件
  HTTP           Axios          API 請求
  構建            Vite 7         快速開發

後端層
  API 框架        FastAPI        高效能 Python API
  資料庫          Firestore      主要資料存儲
  舊系統          Odoo 17        業務邏輯（漸進式淘汰中）

第三方服務
  OCR            GPT-4o (Vision) 模型   名片影像辨識
  通訊            LINE Bot       即時通訊
  認證            Firebase Auth  JWT 認證
  工作表          Google Sheets   專案資料同步

部署
  容器            Docker         環境一致性
  編排            Docker Compose 多容器管理
  反向代理        Caddy          HTTPS + 路由
  雲端            GCP VM         運行環境
```

---

## 7. 設計哲學總結

### 我的技術決策原則

| 面向 | 我的選擇 | 理由 |
|-----|---------|------|
| 資料中心 | Firestore | 實時性 + 無維護 + 自動備份 |
| 關係模型 | Project-centric | 符合真實業務（同人多公司） |
| 快取策略 | 寫入時失效 | 開發友好 + 資料一致性 |
| 前端框架 | React + Ant Design | 生態成熟 + 快速開發 |
| 部署方式 | Docker Compose | 多環境一致 + 易於維護 |

### 我解決問題的思路

1. **先理解業務需求**：為什麼需要「專案為關係中心」？因為同一人常代表多家公司
2. **漸進式改進**：從 GAS → Odoo → Firestore，每次遷移都有明確動機
3. **實用主義**：保留 Odoo 舊系統，不追求完美架構，優先解決業務問題
4. **防禦性設計**：Self-Healing Cache、雙層 fallback 策略

### 未來技術債

```
1. 混合 Odoo + Firestore
   → 長期目標：完全遷移到 Firestore

2. 無事務支持（Firestore 限制）
   → 複雜操作需要「補償事務」設計

3. 權限控制簡化（只有 admin/user）
   → 後期可加入細粒度角色控制
```

---

> 這份文件記錄了我在 45 天內從零建立企業級 CRM 系統的技術思維。
> 每個設計決策都有其業務背景和技術考量，不是為了技術而技術。
