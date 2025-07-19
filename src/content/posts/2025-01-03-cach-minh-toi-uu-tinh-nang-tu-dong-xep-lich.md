---
published: 2025-01-03
title: Cách mình tối ưu tính năng tự động xếp lịch
---

Vừa rồi tự dưng mình có hứng thú và có một vài ý tưởng nên mình đã update lại trang web tự động xếp lịch mà mình đã phát triển từ 2019 dành cho sinh viên trường mình. Sản phẩm mình deploy ở link [này](https://tin-chi-ngosangns.web.app/).

Trong đợt cập nhật này mình thêm tính năng tự động xếp lịch sao cho các lịch ít trùng nhau nhất có thể (dành cho các bạn nào lười xếp như mình).

Vấn đề lớn nhất của tính năng tự động xếp lịch lần này là phải tối ưu tốc độ xử lý, bộ nhớ và giảm bớt số lượng tổ hợp trong không gian mẫu vì số lượng tổ hợp các lớp của tất cả các môn thật sự lớn (khoảng trên 7 triệu tổ hợp cho 10 môn) và ăn hơn 2GB bộ nhớ RAM. Do đó mình đã dành phần lớn thời gian cho việc tối ưu thay vì xây dựng tính năng.

## 1. Tối ưu bộ nhớ của tiết học

Trước đó, mình lưu lịch học của lớp như sau:

```ts
{
    "classCode": string, // Mã lớp
    "schedules": [ // Các lịch học của lớp
        {
            "startDate": number, // Ngày bắt đầu
            "endDate": number, // Ngày kết thúc
            "dayOfWeek": number, // Thứ trong tuần
            "startSession": number, // Tiết bắt đầu
            "endSession": number, // Tiết kết thúc
        }
    ]
}
```

Các lịch trong `schedules` đều đã được sắp xếp từ nhỏ đến lớn theo `startDate`.

Từ cấu trúc trên, để thực hiện tính số tiết học trùng nhau giữa 2 lớp (khác môn) ta cần thực hiện các bước sau:

```
totalOverlap = 0 // Tổng số tiết trùng của tổ hợp

Lặp qua các lịch (schedule) của lớp 1:
 Lặp qua các lịch (schedule) của lớp 2:
  Nếu như startDate và endDate của 2 lịch chồng nhau:
   Nếu dayOfWeek của 2 lịch trùng nhau:
    Nếu startSession và endSession chồng nhau:
     totalOverlap += Số tuần trùng * Số tiết trùng
```

Số tiết trùng được tính như sau:

```js
min(endSession1, endSession2) - max(startSession1, startSession2) + 1
```

Theo mình tìm hiểu thì số bit mặc định của 1 number trong Javascript là 64 bit. Vì vậy số bộ nhớ cần cho việc lưu tiết học là 64 * 4 = 256 (bit). Và để tính ra kết quả cho phép tính trên ta cần 4 phép tính.

Do đó mình đã nảy ra ý tưởng lưu `startSession` và `endSession` dưới dạng bit:

```ts
"schedules": [ // Các lịch học của lớp
    {
        ...
        "sessionBitmask": number, // Tiết học trong ngày (bitmask)
    }
]
```

Ví dụ cho lịch học từ tiết 1 -> 3:

```
1110000000000000
```

Ví dụ từ tiết 7 -> 9:

```
0000001110000000
```

Lưu như trên thì số tiết trùng sẽ được tính như sau:

```ts
let overlapBitmask = bitmask1 & bitmask2
// bitmask1       = 1110000000000000 = 57344
// bitmask2       = 1111110000000000 = 64512
// overlapBitmask = 0001110000000000 = 7168
while(overlapBitmask) { // Đếm số bit 1
    totalOverlap++;
    overlapBitmask >>= 1; // Dịch qua phải 1 bit
}
```

Việc lưu tiết học bằng bit mask đã giảm bộ nhớ từ 256 bit (4 số nguyên) -> 128 bit (2 số nguyên). Mặc dù số lượng phép tính có tăng lên ở đoạn đếm số bit 1 nhưng ta có thể cải thiện bằng cách dùng cache.

```js
if(cache.hasOwnProperty(overlapBitmask)) return cache[overlapBitmask];

localTotalOverlapSession = 0;

while(overlapBitmask) { // Đếm số bit 1
 totalOverlap++;
 localTotalOverlapSession++;
 
 overlapBitmask = overlapBitmask >> 1; // Dịch qua phải 1 bit
}

cache[overlapBitmask] = localTotalOverlapSession;
```

Kết quả là thời gian tính lịch trùng đã giảm đi, còn giảm bao nhiêu thì lười đo qué nhưng trong khoảng chấp nhận được (khoảng +-15s cho 7 triệu tổ hợp)!

## 2. Tối ưu không gian mẫu

Để tính ra tổ hợp tối ưu dựa trên số lượng lịch trùng (tối ưu nhất là số lượng lịch trùng = 0). Ta cần tính ra tất cả các tổ hợp và sau đó tính số lượng lịch trùng cho từng tổ hợp. Nghe thôi đã thấy không ổn rồi.

Do đó thay vì mình tính ra tổ hợp trước rồi mới tính số tiết trùng thì mình sẽ vừa tính số tiết trùng trong lúc tạo tổ hợp. Nếu như một tổ hợp có số lượng tiết trùng vượt quá số lượng cho trước (ở đây mình đặt là tối đa 12 tiết trùng cho một tổ hợp) thì sẽ loại tổ hợp đó không tính tiếp nữa. Về thuật toán mình sử dụng Backtracking bằng cách đệ quy:

```ts
// Hàm tạo tổ hợp
function generateCombination(
  current: number[],
  index: number,
  overlap: number
) {
  // Nếu đã chọn hết tất cả các môn thì thêm tổ hợp này vào danh sách tổ hợp
  if (index === selectedSubjects.length) {
    combinations.push([overlap, ...current]);
    return;
  }
  // Lấy danh sách các lớp của môn học hiện tại
  const classesData = classes[index];
  for (let i = 0; i < classesData.length; i++) {
    let newOverlap = overlap;

    // Lấy lịch học của lớp hiện tại
    const currentClassTimeGrid = timeGrid[index][i];

    // Tính số tiết học bị trùng trong trường hợp thêm lớp này vào tổ hợp
    for (let j = 0; j < index; j++) {
      newOverlap += calculateOverlapBetween2Classes(
        timeGrid[j][current[j]],
        currentClassTimeGrid
      );

      // Nếu số tiết học bị trùng vượt quá ngưỡng thì bỏ qua tổ hợp này
      if (newOverlap > threshold) break;
    }

    // Nếu số tiết học bị trùng vượt quá ngưỡng thì bỏ qua tổ hợp này
    if (newOverlap > threshold) continue;

    current[index] = i; // Lưu lớp được chọn
    
    // Tìm tổ hợp cho môn tiếp theo
    generateCombination(current, index + 1, newOverlap);
  }
}
```

Tạm thời mình làm như thế. Nếu các bạn có ý tưởng nào hay hơn thì comment giúp mình với nhé!

## 3. Trải nghiệm người dùng

Do tính năng tự động xếp lịch nặng về tính toán nên để tránh việc UI thread bị khựng trong khi tính toán thì mình đã chuyển hàm tự động xếp lịch sang thực hiện ở Worker.

```ts
const response = await new Promise(
  (resolve, reject) => {
    try {
      this.calendarWorker.onmessage = (data) => resolve(data);
      this.calendarWorker.postMessage({ ...data });
    } catch (e: unknown) {
      reject(e);
    }
  }
);
```

## 4. Đa luồng

Về vấn đề sử dụng đa luồng, do worker chỉ sử dụng đơn luồng nên mình đã tận dụng multiple worker cho việc tính toán nhưng bị giới hạn ở việc gửi dữ liệu sau khi tính toán.

Cụ thể, mình đã chia các tổ hợp ra các chunk để xử lý song song ở các worker khác nhưng khi các child worker xử lý xong thì cần phải gửi kết quả về parent worker để combine.

Do các worker thực hiện trong các execution context riêng biệt nên khi gửi một message từ worker này sang worker khác thì data phải trải qua bước `serialize` và `deserialize` (như cách ta gửi một POST request với JSON body data). Việc này tốn rất nhiều time và memory cho việc xử lý nên mình đã loại trừ việc sử dụng đa luồng cho đến khi engine có thể hỗ trợ việc chuyển object reference giữa các worker (Hiện tại chỉ hỗ trợ `ArrayBuffer`, `MessagePort` và `ImageBitmap`).

## 5. GPU

Mình cũng đã suy nghĩ đến việc sử dụng GPU cho việc tính toán tổ hợp thông qua thư viện tensorflow.js (WebGL). Nhưng do việc tính toán trên tensor là bao gồm các phép tính song song. Do đó mình bắt buộc phải tính toán toàn bộ tổ hợp trước rồi mới tính số tiết trùng được. Việc này phát sinh rất nhiều phép tính và bộ nhớ do phải tính luôn số tiết trùng trên các tổ hợp có thể dự đoán trước nếu ta thực hiện tuần tự trên một luồng. Ví dụ sau đây sử dụng tensorflow.js để tính toán song song:

```js
const overlaps = [];
for (const chunk of chunkedCombinations) {
    // Gather class indices for each combination
    const classIndices1 = tf.gather(chunk, i, 1);
    const classIndices2 = tf.gather(chunk, j, 1);

    // Get schedules for selected classes
    const selectedSchedules1 = subjectMatrices.gather(classIndices1);
    const selectedSchedules2 = subjectMatrices.gather(classIndices2);

    // Split schedule components
    const [start1, end1, day1, mask1] = tf.split(selectedSchedules1, 4, -1);
    const [start2, end2, day2, mask2] = tf.split(selectedSchedules2, 4, -1);

    // Filter out schedules with start1 or start2 equal to 0
    const validSchedules = tf.logicalAnd(
        tf.notEqual(start1, tf.scalar(0, "int32")),
        tf.notEqual(start2, tf.scalar(0, "int32"))
    );

    const hasOverlap = tf.logicalAnd(
        tf.lessEqual(start1, end2),
        tf.greaterEqual(end1, start2)
    );
    const dayMatch = tf.equal(day1, day2);
    const bitmaskOverlap = tf.bitwiseAnd(mask1.toInt(), mask2.toInt());

    const totalWeeks = tf.ceil(
        tf.div(
            tf.sub(tf.minimum(end1, end2), tf.maximum(start1, start2)),
            tf.scalar(aWeekInMs)
        )
    );

    let bitmaskOverlapShifted = bitmaskOverlap;
    const bitCounts = [];

    for (let i = MIN_SESSION; i <= MAX_SESSION; i++) {
        bitCounts.push(
            tf.bitwiseAnd(
                bitmaskOverlapShifted,
                tf.fill(bitmaskOverlap.shape, 1, "int32")
            )
        );
        bitmaskOverlapShifted = tf.floorDiv(bitmaskOverlapShifted, 2);
    }

    const totalBitCounts = tf.addN(bitCounts);

    // Calculate total overlap
    const pairOverlaps = tf.mul(
        tf.mul(
            tf.mul(tf.cast(hasOverlap, "int32"), tf.cast(dayMatch, "int32")),
            tf.mul(totalWeeks, tf.cast(validSchedules, "int32"))
        ),
        totalBitCounts
    );

    const summedPairOverlaps = tf.sum(pairOverlaps, [1, 2, 3, 4]);
    overlaps.push(summedPairOverlaps);
}
```

Ngoài ra việc tính toán trên GPU còn gây ra việc đơ luôn giao diện của người dùng (do GPU resource bị lấy đi hết để tính toán rồi còn đâu). Mình cũng đã thử một số cách giới hạn resource lại nhưng có vẻ không có hiệu quả.

Do đó việc tính toán trên GPU mình vẫn đang bỏ ngõ tại [đây](https://github.com/ngosangns/tin-chi/blob/master/src/app/app-calendar/calendar2.deprecated.worker.ts).
