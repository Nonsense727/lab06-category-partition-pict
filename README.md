# Лаб 06 — Category-Partition ба PICT (2026)

**Оюутны нэр:** Т. Амармэнд
**Оюутны код:** B232270036

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

## Хэсэг B: PICT — pairwise covering array

**PICT тохиргоо:** PICT-ийг `~/tools/pict` (github.com/microsoft/pict, commit ab76c25)
хаягаас эх кодоос бүтээж `~/tools/pict-bin` замаар ажиллууллаа (macOS дээр sudo
зөвшөөрөгдсөнгүй тул /usr/local/bin/-руу хуулсангүй; нотолгоо: results/pict-build.txt).
`/r` (санамсаргүй seed) хэрэглээгүй тул бүх гаралт үнэмлэхүйц, хязгааргүй.

### B1. Лекцийн browser жишээ

`browser.model` (лекцээс: Content/Cookies = Allow,Restrict,Block; Popups/Addons/Attacks/Forgeries = Yes,No):

- `pict browser.model` (2-way) → **10 мөр** (results/browser-2way.txt)
- `pict browser.model /o:3` (3-way) → **20 мөр** (results/browser-3way.txt)

**2-way хэдэн мөр гарав?** 10 мөр. Лекц дээр гараар 9 мөрөнд багцаж байсан — PICT нэгээр
их, 10 мөр өгсөн. **Яагаад?** PICT greedy алгоритмтай: мөр бүрт хамгийн олон шинэ хосыг
хаахаар нэмэгдүүлж явдаг тул хамгийн цөөнийг (optimal) **заавал** өгдөггүй. Гараар хийсэн
9 мөрийн шийдэл тодорхой зохион байрлалаар бөглөгдсөн (хавтгайруулан); 2-way хамралт
хамгийн багадаа 3 мөрөөр ч багтах зарчимтай боловч бүх параметрийн хослолыг 9-10 мөрөөр
хаах нь гол ёсноо хэвийн — зөвхөн PICT-ийн greedy дүрмийн дагуу нэгээр их мөр шаардлага
болсон.

**3-way хэдэн мөр болов?** 20 мөр. Бүрэн олонлог: 3 × 3 × 2 × 2 × 2 × 2 = **144 хослол**.
20 мөр нь 144-ийн ~14%-д багтаан бүх гурван утны хослолыг хааж байна.

### B2. Шилжүүлгийн моделууд

Гурван хувилбар (бүгд A1-ийн сонголт/утга дээр суурилсан):

| Хувилбар | Файл | Онцлог | Мөр тоо | Happy path | Алдааны тооцоо |
|---|---|---|---|---|---|
| Хязгаарлалтгүй | transfer-nc.model | constraint байхгүй | **13** | 0 | 3 мөр × 1 алдаа, 6 мөр × 2 алдаа, 4 мөр × 3 алдаа (нийт 27 алдааны утга) |
| A2 хязгаарлалттай | transfer.model | R1–R3 (single-error) | **14** | 0 | бүх 14 мөр яг 1 алдаа |
| Сөрөг утгатай | transfer-neg.model | `~` тэмдэглэгээ + R1–R3 | **16** | 2 | 14 мөр × 1 алдаа |

PICT-ийн `IF ... THEN ... AND ...` хэлбэрийн хязгаарлалт бүтээгдэхүүний үүсгэгчинд
амжилттай хэрэгжлээ.

**Гаралтаа уншаад:**

*Happy path (бүх утга хүчинтэй) мөр гарсан уу?* — transfer-nc: **ҮГҮЙ**; transfer: **ҮГҮЙ**;
transfer-neg: **ТИЙМ (2 мөр)** — `Valid Valid Normal Same` ба `Valid Valid Normal Different`.
Үндэслэл: pairwise хамралт зөвхөн хосуудыг хаахад хангалттай. (From=Valid, Amount=Normal)
хосыг (Valid, Missing) мөр хааж байгаа тул бүх утга хүчинтэй нэг мөр хэрэггүй болдог —
PICT happy path-ийг хамралтаас хасаж чадна. Энэ нь category-partition-тэй гол ялгаа:
бидний A3-ийн #1/#2 спецификацийг гараар тодорхойлсон.

*Нэг мөрөнд хэдэн алдаа зэрэг орсон бэ?* — transfer-nc дээр: 3 мөр 1 алдаа, **6 мөр 2
алдаа, 4 мөр 3 алдаа**; transfer дээр: бүх 14 мөр яг 1 алдаа; transfer-neg дээр: 14 мөр
1 алдаа + 2 happy path.

*Олон алдаатай мөр юуг шалгаж байгаа нь тодорхой юу?* — Error masking: нэг мөрөнд хэд хэдэн
буруу утга байвал зөвхөн **давуу эрэмбийн ХАМГИЙН ТҮРҮҮН** алдаа (from → to → amount)
харагдана. Жишээ: `Frozen Missing Zero Different` (transfer-nc 5-р мөр) дээр гурван алдаа
байгаа ч гарах үр дүн `ERROR_FROZEN` — `ERROR_NO_TO`, `ERROR_BAD_AMOUNT` нь хэзээ ч
буцаагдахгүй. Тийм тул олон алдаатай мөр нь "бусад алдааг илрүүлэх" тест биш, зөвхөн
давуу эрэмбийн эхний алдааны error handler-ийг шалгах нэг түвшний тест.

*~ хэрэглэхэд A2-ийн хязгаарлалт хэрэгтэй хэвээр үлдэх үү?* — Туршилтаар: `~` тэмдэглэгээтэй
model-оос R1–R3 дүрмүүдийг хасаад `pict`-ийг ажиллуулбал **яг ижилх 16 мөр** гарлаа.
Тайлбар: `~` нь "нэг мөрөнд хамгийн ихдээ нэг сөрөг утга" дүрмийг PICT-д хэрэгжүүлдэг тул
R1–R3-ийн зорилго (single-error) автоматаар хамаарагддаг. Харин **буруу биш утгууд дундын
хамаарлын** хязгаарлалт (жишээ нь `IF [Currency]="Different" THEN ...`) `~`-ээр хаагддаггүй
тул ийм хамаарлал байвал IF...THEN хязгаарлалт шаардлагатай хэвээр. Бидний model-д
документацийн хувьд A2-ийн дүрмүүдийг хадгалсан.

**Мөрийн тооны харьцуулалт:** 13 (nc) → 14 (constrained) → 16 (~). Constraint нэмэгдсэнгээр
1 мөр НЭМЭГДСЭН — хязгаарлалтаар болзошгүй мөрүүдийг хасахад greedy хосийн хамралтыг
богино хаалттай хаахад илүү мөр шаардлага болсон; `~` нэмэгдсэнгээр 16 болсон (нэг мөрөнд
нэг сөрөг утга гэсэн дүрэм greedy-ийн өрөөнийг бүр богино болгосоноор). Зургийн хувьд:
`~`-тэй model нь A2-ийн **16 системatik комбинацыг яг бүрэн** (happy 2 + single-error 14)
гаргадаг бол constraint-тэй энгийн model 16-ийн 14-ийг авч үздэг (happy path хоёрыг хасаад).

### B2-ийн бодит тест кейсүүд

**TC-01 — happy path** (transfer-neg.txt 2-р мөр: `Valid Valid Normal Same`)
- Setup: from=ACC-1 (үлдэгдэл 100,000 MNT, царцаагаагүй, өдрийн лимитээс 10,000 хэрэглэсэн,
  үлдсэн 90,000); to=ACC-2 (үлдэгдэл 5,000 MNT); currency=MNT
- Оролт: `POST /transfer {"from":"ACC-1","to":"ACC-2","amount":5000,"currency":"MNT"}`
- Oracle: HTTP 201, `{"result":"OK"}`; ACC-1 үлдэгдэл → 95,000, ACC-2 → 10,000

**TC-02 — ERROR_INSUFFICIENT, хөрвүүлэлттэй single error** (transfer-neg.txt 4-р мөр: `Valid Valid ~OverBalance Different`)
- Setup: from=ACC-1 (MNT данс, үлдэгдэл 100,000; ханш 1 USD = 3,400 MNT); to=ACC-2
- Оролт: `{"from":"ACC-1","to":"ACC-2","amount":50,"currency":"USD"}`
  (хөрвүүлсэн дүн = 50 × 3,400 = 170,000 > 100,000 үлдэгдэл)
- Oracle: HTTP 200, `{"result":"ERROR_INSUFFICIENT"}`; үлдэгдэл өөрчлөгдөөгүй
  (IF property: үлдэгдлийг хөрвүүлсний дараах дүнгээр шалгана)

**TC-03 — ERROR_FROZEN, single error** (transfer-neg.txt 13-р мөр: `~Frozen Valid Normal Same`)
- Setup: from=ACC-F (царцаагдсан данс); to=ACC-2
- Оролт: `{"from":"ACC-F","to":"ACC-2","amount":5000,"currency":"MNT"}`
- Oracle: HTTP 200, `{"result":"ERROR_FROZEN"}`; шилжүүлэг гараагүй

**TC-04 — error masking, олон алдаатай мөр** (transfer-nc.txt 5-р мөр: `Frozen Missing Zero Different`)
- Setup: from=ACC-F (царцаагдсан); to=ACC-999 (байхгүй); currency=USD
- Оролт: `{"from":"ACC-F","to":"ACC-999","amount":0,"currency":"USD"}`
- Oracle: HTTP 200, `{"result":"ERROR_FROZEN"}` — **давуу эрэмбэ from → to → amount**-ийн
  дагуу from-ийн алдааны ERROR буцана; `ERROR_NO_TO`, `ERROR_BAD_AMOUNT` хэзээ ч буцаагдахгүй.
  Энэ мөр давуу эрэмбийг семантиктой нотлох тест.

### B3. Гараар (A3) vs PICT — давуу/сул тал

| | Category-Partition (A3) | PICT covering array |
|---|---|---|
| Алдааны семантик | Алдаа бүрт ямар ERROR буцах, давхар нөхцөлд давуу эрэмбээр ямар ERROR гаргахыг урьдчилан тодорхойлно | Семантик мэдэхгүй; мэдээлэлгүй бол алдааны мөрийн oracle тодорхойгүй |
| Happy path | Зорилтоор гаргадаг (A3 #1, #2) | Хамралтаар хасагдах магадлалтай (transfer.txt-д гарсангүй) |
| Хосын (pairwise) хамралт | Гараар бодоход тоо ихсэхэд алдаа гарна, баталгаа байхгүй | Математикаар баталгаажуулна (144 → 20 мөр) |
| Single-error зарчим / хязгаарлалт | Гараар хязгаарлаж, хуваарилдаг | `~` болон IF...THEN дүрмээр автоматжуулна |
| Хэмжээ | 19 спецификац (бүрэн хуваарь) | 13–16 мөр (хамралтын subset) |

## Дүгнэлт

1. Бид хоёр тест сонголтын аргыг `POST /transfer` функцийн дизайн дээр практикт хэрэгжүүллээ:
   category-partition-аар 72 бүрэн хослолоос single-error хязгаарлалт (R1–R3) авч
   16 системatik + 3 давуу эрэмбийн combo = 19 спецификац гаргалаа.
2. Давуу эрэмбийг from → to → amount гэж тогтоож, хэд хэдэн алдаа зэрэг байгаа үед
   буцах ERROR-ийг урьдчилан таамагладаг oracle загварчилж чадлаа.
3. Далд орчны сонголт (өдрийн лимитийн үлдэгдэл, валютын ханш) болон IF/property
   (хөрвүүлсний дараах дүнгээр шалгах) зарчмыг тест setup-д тусгалаа.
4. PICT-ээр pairwise covering array байгуулж, гурван хувилбарын мөрийн тоог (13, 14, 16)
   харьцууллаа: constraint нэмэгдсэнгээр нэг мөр нэмэгдсэн, `~` нэмэгдсэнгээр 16 болсон.
5. Гайхалгүй туршилт: `~` тэмдэглэгээтэй model-оос A2-ийн хязгаарлалтыг хасаад
   ажиллуулбал яг ижил 16 мөр гарлаа — `~` нь single-error зарчмыг автоматаар
   хамааруулдаг тул error-vs-error хязгаарлалт хамаарагүй болсон.
6. Уг хамаарлыг бодит туршилтаар нотолж, харин буруу биш утгууд дундын (non-error)
   хязгаарлалт `~`-ээр хаагдахгүй тул ийм хамаарлал байвал IF...THEN хязгаарлалт
   хэрэгтэй хэвээр гэснийг харууллаа.
7. PICT greedy тул хамгийн цөөнийг заавал өгдөггүй — browser жишээнд лекцийн гараар 9
   мөрөнд багцаасан хаалтыг PICT 10 мөрөөр хаасан; 3-way хамралт бүрэн 144-ийн оронд 20.
8. PICT-ийн хамгийн том сул тал: pairwise шийдэл happy path-ийг ч хасаж чадна
   (transfer.txt-д happy path гарсангүй, харин transfer-neg-д 2 мөр гарсан).
9. Хоёр аргыг хослуулбал: category-partition-ийн семантик + oracle + давуу эрэмбэ PICT-ийн
   хосын хамралтын автоматжуулалт-тай хоёр талаараа бүрэн хамрагдал гардаг
   (transfer-neg-ийн 16 мөр = A2-ийн 16 системatik комбинацийн яг бүрэн хуулбар).
10. Хамгийн их бодол шаардсан алхам: параметр бүрийн утгуудыг [ERROR]/[SINGLE]/[IF]
    ангилж single-error хязгаарлалт гаргах, N=72-оос M=19 болгож хуваарьлах, мөн
    давуу эрэмбээ зөв үндэслэх явдал байлаа; их тооны хослолтой учраас гараар бодох
    нэг бүрчлэн хэцүү байсан.

