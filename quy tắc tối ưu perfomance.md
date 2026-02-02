# 🚀 React Performance Rules - Tránh Re-render Lung Tung

> **Áp dụng cho mọi domain/feature trong project**

---

## 📏 Rule #1: Giới hạn kích thước Hook

> [!CAUTION]
> **Hook return >15 giá trị = NGUY HIỂM**

### ❌ Bad - Hook quá lớn
```typescript
export const useFeature = () => {
  // 30+ state declarations
  const [state1, setState1] = useState()
  const [state2, setState2] = useState()
  // ... 28 more states
  
  // Return 40+ values → RE-RENDER DISASTER
  return {
    state1, setState1,
    state2, setState2,
    // ... 36 more
  }
}
```

**Vấn đề**: Component dùng `const { state1 } = useFeature()` → Khi `state30` thay đổi → Component vẫn re-render!

### ✅ Good - Tách nhỏ hooks

```typescript
// hooks/useFeatureUI.tsx - Chỉ UI states
export const useFeatureUI = () => {
  const [openModal, setOpenModal] = useState(false)
  const [selectedItem, setSelectedItem] = useState()
  return { openModal, setOpenModal, selectedItem, setSelectedItem }
}

// hooks/useFeatureData.tsx - Chỉ data/API
export const useFeatureData = () => {
  const { data, isLoading } = useGetDataQuery()
  const dataSource = useMemo(() => data?.map(...), [data])
  return { data, isLoading, dataSource }
}

// hooks/useFeature.tsx - Orchestrator
export const useFeature = () => {
  const ui = useFeatureUI()
  const data = useFeatureData()
  return { ...ui, ...data }
}
```

**Lợi ích**:
- ✅ UI state thay đổi → Chỉ components dùng UI re-render
- ✅ Data thay đổi → Chỉ components dùng data re-render
- ✅ Code dễ maintain, dễ test

---

## 📏 Rule #2: useCallback - Khi Nào Thực Sự Cần?

> [!WARNING]
> **useCallback/memo KHÔNG phải luôn tốt hơn - Chúng có cost!**

### ⚖️ Trade-offs của useCallback

**Cost của useCallback:**
- ✅ Tạo stable reference → Giảm re-renders
- ❌ Thêm memory để cache function
- ❌ Thêm logic để so sánh dependencies
- ❌ Code phức tạp hơn

**Kết luận**: Chỉ dùng khi **lợi > hại**!

---

### ✅ KHI NÀO NÊN dùng useCallback?

#### 1. Handler được pass xuống Component đã memo

```typescript
// ✅ GOOD - Component đã memo + nhận callback
const FormSearch = memo(({ onSearch }) => {
  return <div>...</div>
})

// Hook
const handleSearch = useCallback(async (data) => {
  await search(data)
}, [search]) // ✅ Cần thiết!

<FormSearch onSearch={handleSearch} />
```

**Lý do**: Nếu không có `useCallback` → `handleSearch` mới mỗi render → `memo` vô dụng!

#### 2. Handler trong dependencies của useEffect/useMemo/useCallback khác

```typescript
// ✅ GOOD - handleFetch trong deps của useEffect
const handleFetch = useCallback(async () => {
  await fetchData()
}, [fetchData])

useEffect(() => {
  handleFetch()
}, [handleFetch]) // ✅ Cần thiết để tránh infinite loop!
```

#### 3. Handler được pass vào Context Provider

```typescript
// ✅ GOOD - Context value thay đổi → Toàn bộ consumers re-render
const handleUpdate = useCallback((data) => {
  setData(data)
}, [])

<FeatureContext.Provider value={{ handleUpdate }}>
  {children}
</FeatureContext.Provider>
```

#### 4. Handler return từ Custom Hook được dùng nhiều nơi

```typescript
// ✅ GOOD - Hook dùng ở 10+ components
export const useFeature = () => {
  const handleSubmit = useCallback(async (values) => {
    await postData(values)
  }, [postData])
  
  return { handleSubmit } // ✅ Nên cache
}
```

---

### ❌ KHI NÀO KHÔNG CẦN useCallback?

#### 1. Handler chỉ dùng trong component đó (không pass xuống)

```typescript
// ✅ GOOD - Không cần useCallback
const MyComponent = () => {
  const handleClick = () => {
    console.log('clicked')
  }
  
  return <button onClick={handleClick}>Click</button>
  // Native DOM element → Không care reference mới
}
```

**Lý do**: Native elements (`<button>`, `<div>`) không so sánh props → useCallback vô ích!

#### 2. Component con CHƯA được memo

```typescript
// ❌ BAD - FormSearch chưa memo → useCallback vô nghĩa
const FormSearch = ({ onSearch }) => { // Không memo!
  return <div>...</div>
}

// Không cần useCallback vì FormSearch vẫn re-render
const handleSearch = (data) => {
  search(data)
}

<FormSearch onSearch={handleSearch} />
```

**Fix đúng**: Memo component TRƯỚC, rồi mới useCallback!

#### 3. Handler đơn giản, dependencies thay đổi liên tục

```typescript
// ❌ BAD - Dependencies đổi mỗi render → useCallback vô dụng
const handleSubmit = useCallback((values) => {
  console.log(searchTerm, filters, page, sortBy) // 4 deps thay đổi liên tục
  submit(values)
}, [searchTerm, filters, page, sortBy])

// ✅ GOOD - Bỏ useCallback đi
const handleSubmit = (values) => {
  console.log(searchTerm, filters, page, sortBy)
  submit(values)
}
```

#### 4. Component render rất nhanh (<1ms)

```typescript
// ❌ OVERKILL - Component đơn giản
const SimpleButton = ({ onClick, label }) => (
  <button onClick={onClick}>{label}</button>
)

// Không cần memo + useCallback cho component này!
```

---

### 🎯 Rule Chính Xác Hơn

> [!TIP]
> **Dùng useCallback KHI VÀ CHỈ KHI:**
> 1. Handler pass xuống component ĐÃ memo
> 2. Handler trong dependencies của hooks khác
> 3. Handler trong Context Provider
> 4. Handler return từ shared custom hook
>
> **Các trường hợp khác**: Đừng dùng! Tốn công vô ích.

---

## 📏 Rule #3: useMemo cho Computed Values

> [!WARNING]
> **Array/Object được tính toán lại mỗi render = Re-render components con**

### ❌ Bad - Tính toán lại mỗi render

```typescript
export const useFeature = () => {
  const { data } = useGetDataQuery()
  
  // ❌ Array mới mỗi render (dù data không đổi)
  const dataSource = data?.content.map((item, idx) => ({
    ...item,
    no: idx + 1
  })) ?? []
  
  // ❌ Object mới mỗi render
  const config = {
    page: 1,
    size: 10
  }
  
  return { dataSource, config }
}
```

**Vấn đề**: Table nhận `dataSource` prop → Mỗi render hook → `dataSource` reference mới → Table re-render (dù data không đổi!)

### ✅ Good - useMemo với dependencies chính xác

```typescript
export const useFeature = () => {
  const { data } = useGetDataQuery()
  const [page, setPage] = useState(1)
  const [pageSize, setPageSize] = useState(10)
  
  // ✅ Chỉ tính lại khi deps thay đổi
  const dataSource = useMemo(() => {
    return data?.content.map((item, idx) => ({
      ...item,
      no: (page - 1) * pageSize + idx + 1
    })) ?? []
  }, [data?.content, page, pageSize]) // ⚠️ Đầy đủ deps!
  
  const config = useMemo(() => ({
    page,
    size: pageSize
  }), [page, pageSize])
  
  return { dataSource, config }
}
```

**Quick Check**: 
- ✅ Computed arrays/objects → Wrap `useMemo`
- ✅ JSX elements → Wrap `useMemo`
- ❌ Primitive values (string, number) → Không cần

---

## 📏 Rule #4: React.memo - Khi Nào Thực Sự Cần?

> [!CAUTION]
> **memo KHÔNG miễn phí - Có cost về memory và comparison!**

### ⚖️ Trade-offs của React.memo

**Cost của memo:**
- ✅ Skip re-render khi props không đổi
- ❌ Thêm memory để lưu component cached
- ❌ So sánh props mỗi lần parent render (shallow comparison)
- ❌ Code boilerplate

**Kết luận**: Chỉ memo khi component **nặng** hoặc re-render **quá nhiều**!

---

### ✅ KHI NÀO NÊN dùng memo?

#### 1. Component render logic phức tạp/nặng

```typescript
// ✅ GOOD - Component có nhiều tính toán
const ComplexTable = ({ data }) => {
  // 1000+ rows, nhiều computed columns
  const processedData = data.map(heavyTransformation)
  return <Table data={processedData} />
}

export default memo(ComplexTable)
```

**Đo lường**: Dùng React DevTools Profiler → Nếu render time >10ms → Nên memo!

#### 2. Component re-render rất nhiều lần

```typescript
// ✅ GOOD - FormSearch re-render mỗi khi parent thay đổi state khác
const FormSearch = ({ onSearch, initialValues }) => {
  return <Form>...</Form>
}

export default memo(FormSearch)
```

**Ví dụ**: Parent có 10 states → FormSearch re-render 10 lần khi bất kỳ state nào đổi

#### 3. Component trong list/map (có nhiều instances)

```typescript
// ✅ GOOD - Render 100+ UserCards → Memo để tránh toàn bộ re-render
const UserCard = memo(({ user, onSelect }) => {
  return <div>{user.name}</div>
})

// Parent
{users.map(user => <UserCard key={user.id} user={user} />)}
```

#### 4. Component nhận props ổn định (ít thay đổi)

```typescript
// ✅ GOOD - config ít khi thay đổi
const Dashboard = memo(({ config }) => {
  return <ComplexUI config={config} />
})
```

---

### ❌ KHI NÀO KHÔNG CẦN memo?

#### 1. Component đơn giản, render rất nhanh

```typescript
// ❌ OVERKILL
const Label = ({ text }) => <span>{text}</span>

export default memo(Label) // Không cần! Cost of memo > cost of re-render
```

**Test**: Render time <1ms → Không cần memo!

#### 2. Props luôn luôn thay đổi

```typescript
// ❌ VÔ ÍCH - children là JSX mới mỗi render
const Wrapper = memo(({ children }) => {
  return <div>{children}</div>
})

<Wrapper>
  <div>Content</div> {/* JSX mới mỗi lần! */}
</Wrapper>
```

#### 3. Component chỉ có 1 instance, không re-render nhiều

```typescript
// ❌ KHÔNG CẦN - Chỉ render 1 lần khi mount
const AppHeader = () => {
  return <header>My App</header>
}

export default memo(AppHeader) // Lãng phí!
```

#### 4. Component đã optimize bằng cách khác

```typescript
// ❌ TRÙNG LẶP
const FormSearch = () => {
  // Đã tách hook, callbacks stable
  const { handleSearch } = useOptimizedSearch()
  return <Form onSubmit={handleSearch} />
}

export default memo(FormSearch) // Có thể không cần nếu props đã stable
```

---

### 🎯 Rule Chính Xác Hơn

> [!TIP]
> **Dùng memo KHI:**
> 1. Component render time >10ms (đo bằng Profiler)
> 2. Component re-render >5 lần mỗi user action
> 3. Component trong list với >20 items
> 4. Props ổn định (không phải inline objects/functions)
>
> **KHÔNG dùng memo khi:**
> - Component đơn giản (<1ms render)
> - Props thay đổi liên tục
> - Chỉ có 1 instance
> - Chưa đo đạc performance

---

### 📊 Measurement Strategy

```typescript
// 1. Thêm log để đo
const MyComponent = ({ data }) => {
  const startTime = performance.now()
  
  // Render logic
  
  useEffect(() => {
    console.log(`Render time: ${performance.now() - startTime}ms`)
  })
  
  return <div>...</div>
}

// 2. Nếu >10ms → Cân nhắc memo
// 3. Nếu <1ms → Không cần memo
```

---

## 📏 Rule #5: Tách Concerns Rõ Ràng

> [!NOTE]
> **1 Hook = 1 Responsibility**

### Pattern chuẩn cho mọi domain:

```
src/pages/[domain]/
├── hooks/
│   ├── useFeatureUI.tsx       # UI states (modals, selections)
│   ├── useFeaturePagination.tsx  # Data fetching & pagination
│   ├── useFeatureActions.tsx  # CRUD handlers (all useCallback)
│   ├── useFeatureDialog.tsx   # Dialog content (useMemo)
│   ├── useFeatureColumn.tsx   # Table columns (useMemo)
│   └── useFeature.tsx         # Main orchestrator
├── components/
│   ├── FormSearch.tsx         # memo ✅
│   ├── FormAdd.tsx            # memo ✅
│   └── FormDetail.tsx         # memo ✅
```

### Hook Orchestrator Template

```typescript
// hooks/useFeature.tsx
export const useFeature = () => {
  // 1. Compose hooks
  const ui = useFeatureUI()
  const data = useFeaturePagination()
  const actions = useFeatureActions({
    // Pass setters from other hooks
    setOpenModal: ui.setOpenModal,
    refetch: data.refetch,
    // ...
  })
  const dialog = useFeatureDialog({
    modeAction: ui.modeAction,
    handleDelete: actions.handleDelete
  })
  
  // 2. Return everything (API không đổi)
  return {
    ...ui,
    ...data,
    ...actions,
    ...dialog
  }
}
```

---

## 🎯 Checklist Trước Khi Commit

Áp dụng cho **MỌI feature mới**:

### Hook Development
- [ ] Hook return <15 values? (Nếu >15 → Tách nhỏ)
- [ ] Mọi handler đều dùng `useCallback`?
- [ ] Dependencies của `useCallback`/`useMemo` đầy đủ?
- [ ] Computed arrays/objects đều dùng `useMemo`?
- [ ] Không có logic business trong component?

### Component Development
- [ ] Form components đã wrap `React.memo`?
- [ ] Components nhận callbacks đã memo?
- [ ] Không destructure hook quá sớm? (Chỉ lấy cần dùng)

### Testing
- [ ] Console.log để check re-renders?
- [ ] React DevTools Profiler để measure?
- [ ] Pagination không trigger toàn bộ components?
- [ ] Modal open/close không trigger form re-render?

---

## 🛠️ Debug Re-render Issues

### Tool 1: Console Log
```typescript
const FormSearch = ({ handleSearch }) => {
  useEffect(() => {
    console.log('🔴 FormSearch rendered')
  })
  // ...
}
```

**Expect**: Chỉ thấy log khi props thực sự thay đổi

### Tool 2: why-did-you-render

```bash
npm install @welldone-software/why-did-you-render
```

```typescript
// src/wdyr.ts
import whyDidYouRender from '@welldone-software/why-did-you-render'

if (process.env.NODE_ENV === 'development') {
  whyDidYouRender(React, {
    trackAllPureComponents: true
  })
}
```

### Tool 3: React DevTools Profiler

1. Mở DevTools → **Profiler** Tab
2. Click **Record** (●)
3. Thực hiện action (search, pagination, modal)
4. Stop recording
5. Xem flame chart → Components nào re-render không cần thiết

---

## 📊 Performance Metrics

### Bad Performance Signals
```
🔴 Click Search → 15+ components re-render
🔴 Pagination → FormSearch + FormAdd re-render
🔴 Modal open/close → Table re-render
🔴 Hook returns 30+ values
🔴 >5 handlers không có useCallback
```

### Good Performance Signals
```
✅ Click Search → Chỉ FormSearch + Table re-render
✅ Pagination → Chỉ Table re-render
✅ Modal open/close → Chỉ Modal content re-render
✅ Hook <15 values, tách theo concerns
✅ 100% handlers dùng useCallback
```

---

## 🎓 Common Mistakes & Fixes

### Mistake 1: Inline Functions

```typescript
// ❌ Bad
<FormSearch 
  handleSearch={(data) => console.log(data)} 
/>

// ✅ Good
const handleSearch = useCallback((data) => {
  console.log(data)
}, [])

<FormSearch handleSearch={handleSearch} />
```

### Mistake 2: Inline Objects

```typescript
// ❌ Bad
<Table pagination={{ page: 1, size: 10 }} />

// ✅ Good
const pagination = useMemo(() => ({ 
  page: 1, 
  size: 10 
}), [])

<Table pagination={pagination} />
```

### Mistake 3: Missing Dependencies

```typescript
// ❌ Bad - Missing 'pageSize' in deps
const dataSource = useMemo(() => {
  return data?.map((item, idx) => ({
    no: idx + 1 + (page - 1) * pageSize
  }))
}, [data, page]) // ⚠️ Missing pageSize!

// ✅ Good
const dataSource = useMemo(() => {
  return data?.map((item, idx) => ({
    no: idx + 1 + (page - 1) * pageSize
  }))
}, [data, page, pageSize])
```

### Mistake 4: Destructuring Hook Sớm

```typescript
// ❌ Bad - Lấy hết mọi thứ
const {
  state1, state2, state3, ... state30
} = useFeature()

// Component re-render khi BẤT KỲ state nào đổi

// ✅ Good - Chỉ lấy cần dùng
const { state1, handleSubmit } = useFeature()

// Hoặc tốt hơn: Pass hook vào, destructure trong component
const hook = useFeature()
return <FormSearch handleSearch={hook.handleSearch} />
```

---

## 💡 Pro Tips

### Tip 1: Group Related States
```typescript
// ❌ Bad
const [openModalAdd, setOpenModalAdd] = useState(false)
const [openModalEdit, setOpenModalEdit] = useState(false)
const [openModalDelete, setOpenModalDelete] = useState(false)

// ✅ Good
const [modalState, setModalState] = useState({
  add: false,
  edit: false,
  delete: false
})

const setOpenModalAdd = (isOpen) => 
  setModalState(prev => ({ ...prev, add: isOpen }))
```

### Tip 2: Lazy State Initialization
```typescript
// ❌ Expensive computation mỗi render
const [data, setData] = useState(expensiveComputation())

// ✅ Chỉ compute lần đầu
const [data, setData] = useState(() => expensiveComputation())
```

### Tip 3: Use Refs for Non-render Values
```typescript
// ❌ State cho value không cần re-render
const [lastApiCall, setLastApiCall] = useState(Date.now())

// ✅ Ref
const lastApiCall = useRef(Date.now())
```

---

## 📝 Summary - Rules Tối Quan Trọng

| # | Rule | Quick Check |
|---|------|-------------|
| 1 | **Hook <15 returns** | `wc -l hooks/useFeature.tsx` |
| 2 | **100% handlers có useCallback** | Search `= async` → Phải có `useCallback` |
| 3 | **Computed values có useMemo** | Search `.map(` → Phải có `useMemo` |
| 4 | **Components có memo** | Form components → `export default memo()` |
| 5 | **Dependencies đầy đủ** | ESLint warning `react-hooks/exhaustive-deps` |

---

## 🚦 Implementation Priority

### 🔴 High Priority (Làm ngay)
- Hook >20 returns → Tách ngay
- Handlers không `useCallback` → Fix ngay
- Table/List không `useMemo` dataSource → Fix ngay

### 🟡 Medium Priority (Sprint sau)
- Components chưa memo → Thêm dần
- Inline functions/objects → Refactor dần

### 🟢 Low Priority (Nice to have)
- Micro-optimizations
- Advanced memoization strategies

---

**Áp dụng rules này → Tránh 90% re-render issues ngay từ đầu! 🎯**
