# 餐飲分單 Firestore 規則

餐飲分單使用獨立的 `diningSessions` 集合，聚會分帳仍沿用 `meetings`、`expenses` 和 `transfers`。折扣分配設定使用 `discountAllocation`（`proportional`、`equal`、`person`）及 `discountRecipient`。舊餐單沒有這兩個欄位時會沿用按比例分攤。部署新版前，請把以下 helper functions 和 `match /diningSessions/{diningId}` 區塊加入 Firebase Console 的既有 Firestore Rules；不要以此片段取代原有規則。將內容放在既有 `match /databases/{database}/documents` 區塊內、最終的右大括號之前。

```firestore
function validDiningAdjustment(mode, value) {
  return (mode == 'percent' && value is int && value >= 0 && value <= 10000)
    || (mode == 'amount' && value is int && value >= 0 && value <= 100000000);
}

function validDiningDiscountAllocation(data) {
  let hasAllocation = data.keys().hasAny(['discountAllocation']);
  let hasRecipient = data.keys().hasAny(['discountRecipient']);
  return (!hasAllocation && !hasRecipient)
    || (hasAllocation
      && data.discountAllocation == 'proportional'
      && (!hasRecipient || data.discountRecipient == null))
    || (hasAllocation
      && data.discountAllocation == 'equal'
      && (!hasRecipient || data.discountRecipient == null))
    || (hasAllocation
      && data.discountAllocation == 'person'
      && hasRecipient
      && data.discountRecipient is string
      && data.participants.hasAny([data.discountRecipient]));
}

function validDiningSession(data) {
  return data.keys().hasAll([
      'name', 'participants', 'payer', 'serviceMode', 'serviceValue',
      'discountMode', 'discountValue', 'archived', 'createdAt'
    ])
    && data.keys().hasOnly([
      'name', 'participants', 'payer', 'serviceMode', 'serviceValue',
      'discountMode', 'discountValue', 'discountAllocation', 'discountRecipient',
      'archived', 'createdAt'
    ])
    && data.name is string && data.name.size() > 0 && data.name.size() <= 100
    && data.participants is list && data.participants.size() <= 100
    && (data.payer == null || (data.payer is string && data.participants.hasAny([data.payer])))
    && validDiningAdjustment(data.serviceMode, data.serviceValue)
    && validDiningAdjustment(data.discountMode, data.discountValue)
    && validDiningDiscountAllocation(data)
    && data.archived is bool
    && data.createdAt is int && data.createdAt > 0;
}

function diningSessionExists(diningId) {
  return exists(/databases/$(database)/documents/diningSessions/$(diningId));
}

function diningSessionOpen(diningId) {
  return diningSessionExists(diningId)
    && get(/databases/$(database)/documents/diningSessions/$(diningId)).data.archived == false;
}

function validDiningItem(diningId) {
  let parent = get(/databases/$(database)/documents/diningSessions/$(diningId)).data;
  return request.resource.data.keys().hasAll(['amount', 'responsible', 'createdAt'])
    && request.resource.data.keys().hasOnly(['amount', 'responsible', 'createdAt'])
    && request.resource.data.amount is int
    && request.resource.data.amount > 0
    && request.resource.data.amount <= 100000000
    && request.resource.data.responsible is list
    && request.resource.data.responsible.size() > 0
    && request.resource.data.responsible.size() <= 100
    && parent.participants.hasAll(request.resource.data.responsible)
    && request.resource.data.createdAt is int
    && request.resource.data.createdAt > 0;
}

match /diningSessions/{diningId} {
  // 餐單連結持有人可讀寫；無法列出所有餐單。
  allow get: if true;
  allow list: if false;
  allow create: if validDiningSession(request.resource.data);
  allow update: if resource.data.archived == false
    && request.resource.data.diff(resource.data).affectedKeys().hasOnly([
      'name', 'participants', 'payer', 'serviceMode', 'serviceValue',
      'discountMode', 'discountValue', 'discountAllocation', 'discountRecipient', 'archived'
    ])
    && validDiningSession(request.resource.data);
  allow delete: if false;

  match /items/{itemId} {
    allow read: if diningSessionExists(diningId);
    allow create: if diningSessionOpen(diningId) && validDiningItem(diningId);
    allow update: if diningSessionOpen(diningId)
      && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['amount', 'responsible'])
      && request.resource.data.createdAt == resource.data.createdAt
      && validDiningItem(diningId);
    allow delete: if diningSessionOpen(diningId);
  }
}
```

目前餐單連結是編輯連結：任何持有連結的人都能查看和共同編輯；不要在公開位置張貼。`serviceValue` 百分比以 basis points 儲存（10000 = 100%），實際金額與餐點金額以分儲存。
