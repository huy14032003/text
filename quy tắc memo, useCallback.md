# 🚨 QUAN TRỌNG: Đừng Lạm Dụng useCallback và memo!

> [!CAUTION]
> **useCallback/memo CÓ COST! Không phải lúc nào cũng tốt hơn.**

## ❌ Myth: "Càng nhiều useCallback/memo càng tốt"

### Sự thật:

```typescript
// ❌ OVERUSE - Lãng phí, code phức tạp
const SimpleComponent = memo(({ onClick }) => {
  const handleClick = useCallback(() => {
    onClick()
  }, [onClick])
  
  const style = useMemo(() => ({ color: 'red' }), [])
  
  return <button onClick={handleClick} style={style}>Click</button>
})
```

**Chi phí:**
- Memory để cache 3 thứ (component, function, object)
- CPU để compare dependencies và props
- Code khó đọc hơn

**Lợi ích:** = 0 (component render <1ms, không re-render nhiều lần)

---

## ✅ Approach Đúng: Measure First, Optimize Later

### Quy trình 3 bước:

```
1. MẶC ĐỊNH: KHÔNG dùng useCallback/memo
   ↓
2. ĐO LƯỜNG: Dùng React DevTools Profiler
   ↓
3. TỐI ƯU: Chỉ optimize chỗ THỰC SỰ chậm
```

### Khi nào optimize?

| Metric | Không cần | Cân nhắc | Phải optimize |
|--------|-----------|----------|---------------|
| Render time | <1ms | 1-10ms | >10ms |
| Re-renders/action | <3 | 3-10 | >10 |
| Component instances | 1-5 | 5-20 | >20 |

---

## 🎯 Decision Tree: Có nên useCallback?

```
Handler được pass xuống component?
├─ KHÔNG → ❌ Không dùng useCallback
└─ CÓ
   └─ Component đó đã memo?
      ├─ KHÔNG → ❌ Không dùng useCallback
      |           (Memo component trước)
      └─ CÓ
         └─ Component render >10ms?
            ├─ KHÔNG → ❌ Không cần
            └─ CÓ → ✅ Dùng useCallback
```

---

## 🎯 Decision Tree: Có nên memo component?

```
Component render time?
├─ <1ms → ❌ Không memo
└─ >1ms
   └─ Re-render bao nhiêu lần/action?
      ├─ <3 lần → ❌ Không memo
      └─ >3 lần
         └─ Props có thay đổi mỗi render?
            ├─ CÓ (inline objects/functions) → ❌ Fix props trước
            └─ KHÔNG → ✅ Memo component
```

---

## 📝 Checklist Trước Khi useCallback/memo

### Trước khi thêm useCallback:
- [ ] Đã đo render time của component nhận handler?
- [ ] Component đó đã được memo?
- [ ] Dependencies có stable không? (không phải objects/arrays mới)
- [ ] Handler có thực sự được re-use?

### Trước khi memo component:
- [ ] Đã đo render time? (>10ms?)
- [ ] Đã đếm re-renders? (>5 lần/action?)
- [ ] Props có stable không? (không inline)
- [ ] Đã thử optimize bằng cách khác? (tách components, lazy load)

---

## 💡 Practical Guidelines cho Project Này

### ✅ LUÔN LUÔN optimize:

1. **Custom hooks return handlers**
   ```typescript
   // Hook dùng ở nhiều nơi → Nên useCallback
   export const useFeature = () => {
     const handleSubmit = useCallback(...)
     return { handleSubmit }
   }
   ```

2. **Form components**
   ```typescript
   // Form thường nặng → Nên memo
   export default memo(FormSearch)
   export default memo(FormAdd)
   ```

3. **Table/List components**
   ```typescript
   // Render nhiều rows → Nên memo
   export default memo(DataTable)
   ```

### ❌ THƯỜNG KHÔNG CẦN optimize:

1. **Handlers chỉ dùng local**
   ```typescript
   // Native button → Không cần useCallback
   const handleClick = () => { ... }
   <button onClick={handleClick} />
   ```

2. **Simple display components**
   ```typescript
   // <1ms render → Không memo
   const Label = ({ text }) => <span>{text}</span>
   ```

3. **Layout components**
   ```typescript
   // Render 1 lần → Không memo
   const Header = () => <header>...</header>
   ```

---

## 🔍 Real Example: Khi Nào Thêm useCallback

### Scenario: FormSearch component

```typescript
// Step 1: Viết code đơn giản trước
const FormSearch = ({ onSearch }) => {
  return <Form onSubmit={onSearch} />
}

const ParentPage = () => {
  const handleSearch = (data) => {
    search(data)
  }
  
  return <FormSearch onSearch={handleSearch} />
}
```

**Đo lường:**
- FormSearch render time: 15ms ✅ (nặng)
- Parent có 5 states → FormSearch re-render 5 lần ✅ (nhiều)

**Decision:**
```typescript
// Step 2: Memo component trước
const FormSearch = memo(({ onSearch }) => {
  return <Form onSubmit={onSearch} />
})

// Step 3: Thêm useCallback
const ParentPage = () => {
  const handleSearch = useCallback((data) => {
    search(data)
  }, [search]) // ✅ Giờ memo mới có tác dụng!
  
  return <FormSearch onSearch={handleSearch} />
}
```

**Kết quả:**
- FormSearch chỉ re-render khi onSearch thực sự thay đổi ✅
- Giảm 4/5 re-renders không cần thiết ✅

---

## 📊 Performance Budget

Cho project này, áp dụng theo mức độ:

### High Priority (Luôn optimize):
- ✅ Custom hooks với >5 consumers
- ✅ Form components (Search, Add, Edit)
- ✅ Table/List components với >20 rows
- ✅ Modal/Dialog content components

### Medium Priority (Optimize khi cần):
- ⚠️ Components với render time >10ms
- ⚠️ Components re-render >5 lần/action
- ⚠️ Components trong loops

### Low Priority (Thường không cần):
- ❌ Simple display components
- ❌ Layout components
- ❌ Components render 1 lần
- ❌ Components <1ms render time

---

## 🎓 Lessons Learned từ Project Này

### ✅ Đã làm đúng:
1. Tách hooks lớn (40+ returns) → 5 hooks nhỏ
2. useCallback cho handlers trong shared hooks
3. memo cho Form components nặng
4. useMemo cho computed dataSource

### ⚠️ Có thể bỏ:
1. memo cho components đơn giản (<1ms)
2. useCallback cho handlers không pass xuống
3. useMemo cho primitive computations

### 🎯 Nguyên tắc vàng:
**"Optimize vì ĐO ĐƯỢC, không vì SỢ"**

---

## 🛠️ Tools để Đo Lường

### 1. React DevTools Profiler
```
DevTools → Profiler → Record → Thực hiện action → Stop
→ Xem component nào render lâu
→ Xem component nào re-render nhiều nhất
```

### 2. Console Performance
```typescript
const MyComponent = (props) => {
  const renderCount = useRef(0)
  renderCount.current++
  
  console.log(`MyComponent rendered ${renderCount.current} times`)
  
  return <div>...</div>
}
```

### 3. Performance API
```typescript
const start = performance.now()
// Render logic
console.log(`Render time: ${performance.now() - start}ms`)
```

---

## 📝 Summary

> [!NOTE]
> **TL;DR:**
> - ❌ Đừng useCallback/memo mọi thứ
> - ✅ Đo lường trước, optimize sau
> - ✅ Chỉ optimize chỗ THỰC SỰ chậm
> - ✅ Biết khi nào KHÔNG CẦN optimize quan trọng hơn

**Cost vs Benefit:**
```
useCallback/memo có ích KHI:
  Benefit (giảm re-render time) > Cost (memory + comparison)

useCallback/memo LÃNG PHÍ KHI:
  Component render <1ms
  Hoặc props thay đổi liên tục
  Hoặc chỉ re-render 1-2 lần
```
