# Лаб 06 — Category-Partition ба PICT (2026)

**Оюутны нэр:** [өөрийн нэрээ бичнэ үү]
**Оюутны код:** [код]

F.CSA313 — Программ хангамжийн чанарын баталгаа ба тест (2026)

Тестлэх функц (код байхгүй — зөвхөн тодорхойлолт, shift-left дизайн):

```
POST /transfer  { "from": ID, "to": ID, "amount": NUM, "currency": CODE }
```

Дүрмүүд: from данс байх ёстой, царцаагдаагүй; to данс байх ёстой, from-оос өөр;
amount > 0, үлдэгдлээс хэтрэхгүй, өдрийн лимитээс хэтрэхгүй (бүгдийг хөрвүүлсний
дамжих дүнгээр); currency өөр бол ханш хөрвүүлэлт. Амжилт: `201 {"result":"OK"}`,
оролтын алдаа: `200 {"result":"ERROR_..."}`.

---

## Хэсэг A: Category-Partition гараар

### A1. Сонголт ба төлөөлөх утгууд

| Параметр | Сонголт | Төлөөлөх утга | Тайлбар |
|---|---|---|---|
| from | Valid | ACC-1 | данс байгаа, царцаагаагүй, үлдэгдэл 100,000 MNT |
| from | Frozen | ACC-F | данс байгаа, гэхдээ царцаагдсан |
| from | Missing | ACC-999 | ийм данс байхгүй |
| to | Valid | ACC-2 | данс байгаа, from-оос өөр |
| to | Missing | ACC-888 | ийм данс байхгүй |
| to | SameAsFrom | ACC-1 | to == from |
| amount | Zero | 0 | amount ≤ 0 (тэг утгаар төлөөлнэ) |
| amount | Normal | 5,000 | 0 < amount; хөрвүүлсэн дүн үлдэгдэл болон үлдсэн лимитэд багтана |
| amount | OverBalance | 150,000 | хөрвүүлсэн дүн үлдэгдлээс (100,000) их |
| amount | OverLimit | 30,000 | хөрвүүлсэн дүн үлдэгдэлд багтана, гэхдээ үлдсэн өдрийн лимитээс (10,000) их |
| currency | Same | MNT | дансны валюттай адил, хөрвүүлэлтгүй |
| currency | Different | USD | данснаас өөр валют (1 USD = 3,400 MNT ханштай) |

**Далд орчны сонголтууд** (environment state — оролт биш, тест setup-тайгаар
зохион байгуулна):

- **Өдрийн лимитийн үлдэгдэл** (үндсэн асуулт: лимит хэр дүүрсэн байсан бэ?) —
  бүрэн, хагас, эсвэл дууссан. OverLimit утгыг "лимит дууссан" төлөвтэй
  ингэж туршаад зохион байгуулна: үлдсэн лимит = 0, amount = 30,000.
- **Валютын ханш** — currency = Different үед хөрвүүлэлт хийх ханш байх ёстой.
  Ханш байхгүй тохиолдолын алдааны код спецификац дээр заагдаагүй тул тестийн
  сонголтод хамарсангүй.

Хүний дансны үлдэгдэл нь amount-ийн утгуудын утгалагаар (Normal / OverBalance)
шүүж төлөөлөгдсөн.

### A2. Хязгаарлалтууд

| Утга | Тэмдэглэгээ | Үндэслэл |
|---|---|---|
| from=Valid | [SINGLE] | Бусад бүх утга хүчинтэй single тестийн суурь мөр |
| from=Frozen | [ERROR] ERROR_FROZEN | Царцаагдсан данснаас шилжүүлэг хийгдэх ёсгүй |
| from=Missing | [ERROR] ERROR_NO_FROM | Данс байх ёстой — байхгүй данс буцаана |
| to=Valid | [SINGLE] | single тестийн суурь мөр |
| to=Missing | [ERROR] ERROR_NO_TO | Хүргэх данс байх ёстой |
| to=SameAsFrom | [ERROR] ERROR_SAME_ACCOUNT | to == from байж болохгүй |
| amount=Normal | [SINGLE] | single тестийн суурь мөр |
| amount=Zero | [ERROR] ERROR_BAD_AMOUNT | amount > 0 ёстой |
| amount=OverBalance | [ERROR] ERROR_INSUFFICIENT | Хөрвүүлсэн дүн үлдэгдлээс хэтрэхгүй |
| amount=OverLimit | [ERROR] ERROR_LIMIT | Хөрвүүлсэн дүн өдрийн лимитээс хэтрэхгүй |
| currency=Same | [SINGLE] | Хөрвүүлэлтгүй үндсэн зам |
| currency=Different | [IF/property] | Валют өөр бол ханш хөрвүүлэлт хийгдэж, үлдэгдэл/лимитийг **хөрвүүлсний дараах** дүнгээр шалгана (тэгэхдээ шилжүүлгийн гүйцэтгэл нь дансны валютаар) |

Хязгаарлалтын дүрмүүд (single-error зарчим):

```
R1: IF [from] IN {Frozen, Missing} THEN [to] = Valid AND [amount] = Normal
R2: IF [to]   IN {Missing, SameAsFrom} THEN [from] = Valid AND [amount] = Normal
R3: IF [amount] IN {Zero, OverBalance, OverLimit} THEN [from] = Valid AND [to] = Valid
```

Үндэслэл: нэг мөрөнд зөвхөн нэг буруу утга байхаар хязгаарласнаар буцаах ERROR
нь тухайн хохиролтой цор ганц холбогдох бөгөөд error masking-ийг (бусад алдаа нуугддаг
үзэгдлийг) систематик тестээс хасаж, зөвхөн зохиомол давхар combo-оор давуу эрэмбийг
шалгана.

**N → M тооцоо:**

```
N = 3 (from) × 3 (to) × 4 (amount) × 2 (currency) = 72

R1–R3-ийн дараа:
  from ∈ {Frozen, Missing}:              2 × 1 × 1 × 2 = 4
  from=Valid, to ∈ {Missing, SameAsFrom}: 1 × 2 × 1 × 2 = 4
  from=Valid, to=Valid, amount burau:     1 × 1 × 3 × 2 = 6
  from=Valid, to=Valid, amount=Normal:    1 × 1 × 1 × 2 = 2
  --------------------------------------------------
  Систематик тест = 16
  + Давуу эрэмбийн 3 зохиомол combo (#17–#19)
  ==================================
  72 → 16 → 19
```

currency нь алдааны утга биш (утга хоёултаа зөв) тул R1–R3-д ороогүй; тийм тул
currency=Same/Different хоёуланг нь бүх мөрөнд хосолж турхийн.

### A3. Спецификацийн хүснэгт

**Давуу эрэмбэ: from → to → amount.**
Үндэслэл: спецификац дээрх ERROR кодуудын дэс дараалал ба логик — шилжүүлэг
хийх гэж байна гэдгийг хэвлэхийн өмнө эхлээд from дансны хүчинтэй байдлыг шалгадаг
тул хэд хэдэн алдаа зэрэг байвал буцах ERROR нь **хамгийн түрүүн зөрчилдсөн нөхцөл**
болно (error masking: эхний алдаа бусдыгаа нууна).

| № | from | to | amount | currency | Хүлээгдэж буй үр дүн | Тэмдэглэл |
|---|---|---|---|---|---|---|
| 1 | Valid | Valid | Normal | Same | 201 {"result":"OK"} | happy path |
| 2 | Valid | Valid | Normal | Different | 201 {"result":"OK"} | happy path + ханш хөрвүүлэлт (IF) |
| 3 | Frozen | Valid | Normal | Same | 200 {"result":"ERROR_FROZEN"} | single error |
| 4 | Missing | Valid | Normal | Same | 200 {"result":"ERROR_NO_FROM"} | single error |
| 5 | Frozen | Valid | Normal | Different | 200 {"result":"ERROR_FROZEN"} | single error; frozen шалгах нь хөрвүүлэлтээс өмнө |
| 6 | Missing | Valid | Normal | Different | 200 {"result":"ERROR_NO_FROM"} | single error |
| 7 | Valid | Missing | Normal | Same | 200 {"result":"ERROR_NO_TO"} | single error |
| 8 | Valid | SameAsFrom | Normal | Same | 200 {"result":"ERROR_SAME_ACCOUNT"} | single error |
| 9 | Valid | Missing | Normal | Different | 200 {"result":"ERROR_NO_TO"} | single error |
| 10 | Valid | SameAsFrom | Normal | Different | 200 {"result":"ERROR_SAME_ACCOUNT"} | single error |
| 11 | Valid | Valid | Zero | Same | 200 {"result":"ERROR_BAD_AMOUNT"} | single error |
| 12 | Valid | Valid | OverBalance | Same | 200 {"result":"ERROR_INSUFFICIENT"} | single error |
| 13 | Valid | Valid | OverLimit | Same | 200 {"result":"ERROR_LIMIT"} | single error; setup: өдрийн лимит дууссан |
| 14 | Valid | Valid | Zero | Different | 200 {"result":"ERROR_BAD_AMOUNT"} | single error; amount ≤ 0 шалгах нь хөрвүүлэлтээс өмнө |
| 15 | Valid | Valid | OverBalance | Different | 200 {"result":"ERROR_INSUFFICIENT"} | single error; **хөрвүүлсөн** дүн үлдэгдлээс их (IF property) |
| 16 | Valid | Valid | OverLimit | Different | 200 {"result":"ERROR_LIMIT"} | single error; хөрвүүлсөн дүн лимитээс их (IF property) |
| 17 | Frozen | Missing | Zero | Different | 200 {"result":"ERROR_FROZEN"} | давхар нөхцөл; давуу эрэмбэ: from → to → amount |
| 18 | Missing | SameAsFrom | Normal | Same | 200 {"result":"ERROR_NO_FROM"} | давхар нөхцөл; from нь to-оос өмнө |
| 19 | Valid | Missing | OverBalance | Same | 200 {"result":"ERROR_NO_TO"} | давхар нөхцөл; to нь amount-аас өмнө |

(*Хэсэг B — PICT covering array — доор хичээл*)
