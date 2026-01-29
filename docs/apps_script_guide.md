```javascript
function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);

    const houseId = String(data.house_id || "").trim();
    const reqIn = new Date(data.check_in);
    const reqOut = new Date(data.check_out);

    // Валидация дат
    if (isNaN(reqIn.getTime()) || isNaN(reqOut.getTime())) {
      throw new Error("Invalid date format. Please use YYYY-MM-DD format");
    }

    if (reqOut <= reqIn) {
      throw new Error("check_out must be after check_in");
    }

    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sheet = ss.getSheetByName('Bookings');
    const values = sheet.getDataRange().getValues();

    const headers = values[0].map(h => String(h).trim());
    const idxHouse = headers.indexOf('house_id');
    const idxIn = headers.indexOf('check_in');
    const idxOut = headers.indexOf('check_out');

    if (idxHouse === -1 || idxIn === -1 || idxOut === -1) {
      return ContentService
        .createTextOutput(JSON.stringify({
          error: "Headers not found. Need: house_id, check_in, check_out"
        }))
        .setMimeType(ContentService.MimeType.JSON);
    }

    let busy = false;
    let existingCheckIn = null;
    let existingCheckOut = null;

    for (let i = 1; i < values.length; i++) {
      const row = values[i];
      if (String(row[idxHouse]).trim() !== houseId) continue;

      const existingIn = new Date(row[idxIn]);
      const existingOut = new Date(row[idxOut]);

      // Пересечение периодов:
      if (existingIn < reqOut && existingOut > reqIn) {
        busy = true;
        
        // Форматируем даты в YYYY-MM-DD
        const formatDate = (date) => {
          const year = date.getFullYear();
          const month = String(date.getMonth() + 1).padStart(2, '0');
          const day = String(date.getDate()).padStart(2, '0');
          return `${year}-${month}-${day}`;
        };
        
        existingCheckIn = formatDate(existingIn);
        existingCheckOut = formatDate(existingOut);
        break;
      }
    }

    // Форматируем запрошенные даты для ответа
    const formatDate = (date) => {
      const year = date.getFullYear();
      const month = String(date.getMonth() + 1).padStart(2, '0');
      const day = String(date.getDate()).padStart(2, '0');
      return `${year}-${month}-${day}`;
    };

    const result = {
      house_id: houseId,
      check_in: formatDate(reqIn),
      check_out: formatDate(reqOut),
      available: !busy
    };

    // Добавляем даты конфликтующего бронирования, если есть
    if (busy) {
      result.existing_check_in = existingCheckIn;
      result.existing_check_out = existingCheckOut;
      result.message = "Property is already booked for these dates";
    } else {
      result.message = "Property is available for booking";
    }

    return ContentService
      .createTextOutput(JSON.stringify(result))
      .setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    return ContentService
      .createTextOutput(JSON.stringify({
        error: error.message,
        available: false
      }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```
