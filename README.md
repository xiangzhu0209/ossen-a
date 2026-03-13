// 引用 Firebase Functions 和 Admin SDK
const functions = require("firebase-functions");
const admin = require("firebase-admin");

// 初始化 Firebase Admin，讓 Function 可以存取其他 Firebase 服務
admin.initializeApp();

/**
 * 新增一筆員工績效紀錄
 * 這是一個 "Callable Function"，可以從您的 App 客戶端安全地呼叫。
 *
 * @param {object} data - 從客戶端傳來的資料。
 * @param {string} data.employeeId - 員工的 ID。
 * @param {string} data.ruleId - 績效規則的 ID。
 * @param {string} data.notes - 管理員填寫的備註 (可選)。
 * @param {object} context - 包含使用者認證資訊的上下文。
 *
 * @returns {Promise<object>} - 回傳操作結果。
 */
exports.addPerformanceRecord = functions.https.onCall(async (data, context) => {
  // 1. 驗證呼叫者是否為已登入的使用者
  // 為了簡化，我們先假設任何登入者都可操作，
  // 未來您可以加入角色檢查，確保只有 'admin' 能執行。
  if (!context.auth) {
    throw new functions.https.HttpsError(
      "unauthenticated",
      "您必須登入才能執行此操作。"
    );
  }

  const { employeeId, ruleId, notes } = data;
  const db = admin.firestore();

  // 2. 驗證傳入的參數是否齊全
  if (!employeeId || !ruleId) {
    throw new functions.https.HttpsError(
      "invalid-argument",
      "缺少必要的參數：員工 ID (employeeId) 或 規則 ID (ruleId)。"
    );
  }

  try {
    // 3. 從 PerformanceRules 集合中獲取規則的詳細資訊 (特別是分數)
    const ruleDoc = await db.collection("PerformanceRules").doc(ruleId).get();
    if (!ruleDoc.exists) {
      throw new functions.https.HttpsError("not-found", "找不到指定的績效規則。");
    }
    const ruleData = ruleDoc.data();
    const points = ruleData.points; // 取得該規則對應的分數

    // 4. 準備要寫入資料庫的新紀錄
    const newRecord = {
      employeeId: employeeId, // 關聯的員工
      ruleId: ruleId, // 關聯的規則
      points: points, // 當時的分數
      description: ruleData.description, // 當時的規則描述
      notes: notes || "", // 備註，如果沒有提供則為空字串
      recordDate: admin.firestore.FieldValue.serverTimestamp(), // 使用伺服器時間作為紀錄時間
      recordedBy: context.auth.uid, // 記錄是哪個管理者新增的
    };

    // 5. 將新紀錄寫入 PerformanceRecords 集合
    const recordRef = await db.collection("PerformanceRecords").add(newRecord);

    console.log(`成功新增績效紀錄，ID 為: ${recordRef.id}`);

    // 6. 回傳成功訊息
    return {
      status: "success",
      message: "績效紀錄已成功新增！",
      recordId: recordRef.id,
    };
  } catch (error) {
    console.error("新增績效紀錄時發生錯誤:", error);
    throw new functions.https.HttpsError(
      "internal",
      "處理請求時發生未知的伺服器錯誤。"
    );
  }
});
