# Date and Time API

## The Time Line

### `Instant`

- __Instant__ — a point on the time line; nanosecond precision
- __Epoch__ — midnight January 1, 1970 UTC (same as UNIX/POSIX)
- Range: `Instant.MIN` (~1 billion years ago) to `Instant.MAX` (year 1,000,000,000)
- Java's time scale uses 86,400 seconds per day and matches official time at noon each day

```java
Instant now = Instant.now();
Instant later = now.plusSeconds(60);
Instant earlier = now.minusMillis(500);
```

### `Duration`

- __Duration__ — amount of time between two instants; stores seconds (`long`) + nanoseconds (`int`)
- Overflow risk with nanoseconds: `long` holds ~300 years of nanoseconds

```java
Duration elapsed = Duration.between(start, end);
long millis = elapsed.toMillis();
long nanos = elapsed.toNanos();
long seconds = elapsed.toSeconds();
long minutes = elapsed.toMinutes();
long hours = elapsed.toHours();
long days = elapsed.toDays();

// Part extraction
int secPart = elapsed.toSecondsPart();
int minPart = elapsed.toMinutesPart();

// Arithmetic
Duration d2 = elapsed.multipliedBy(10);
Duration d3 = elapsed.dividedBy(2);
Duration d4 = elapsed.negated();
Duration d5 = elapsed.plus(Duration.ofSeconds(30));
Duration d6 = elapsed.minus(Duration.ofMillis(100));

elapsed.isZero();
elapsed.isNegative();
```

- `Instant` and `Duration` are __immutable__ — all methods return new instances

---

## Local Dates

### `LocalDate`

- __LocalDate__ — date with year, month, day; no time zone; no time of day
- Months are 1-based (not 0-based like `java.util.Date`)

```java
LocalDate today = LocalDate.now();
LocalDate d = LocalDate.of(1903, 6, 14);
LocalDate d = LocalDate.of(1903, Month.JUNE, 14);

d.plusDays(255)
d.minusWeeks(2)
d.plusMonths(1)
d.plusYears(1)
d.withDayOfMonth(1)
d.withMonth(12)
d.withYear(2025)

d.getDayOfMonth()   // 1–31
d.getDayOfYear()    // 1–366
d.getDayOfWeek()    // DayOfWeek enum; MONDAY=1, SUNDAY=7
d.getMonth()        // Month enum
d.getMonthValue()   // 1–12
d.getYear()

d.isBefore(other)
d.isAfter(other)
d.isLeapYear()

// Stream of dates
start.datesUntil(endExclusive)                  // step = 1 day
start.datesUntil(endExclusive, Period.ofMonths(1))
```

> [!NOTE]
> Adding one month to January 31 yields February 28/29 (last valid day), not an exception.

### `Period`

- __Period__ — elapsed years, months, days; used for calendar-aware arithmetic

```java
Period p = Period.of(1, 2, 3);         // 1 year, 2 months, 3 days
Period p = Period.ofYears(1);
Period p = Period.ofMonths(6);
Period p = Period.ofDays(10);

p.getYears()
p.getMonths()
p.getDays()

LocalDate next = today.plus(Period.ofDays(7));   // correct across DST
// vs Duration.ofDays(7) which adds 7 * 86400 seconds — wrong across DST
```

- `independenceDay.until(christmas)` — returns `Period` (months + days)
- `independenceDay.until(christmas, ChronoUnit.DAYS)` — returns total days as `long`

### Partial Dates

- `MonthDay` — Eg - December 25 without a year
- `YearMonth` — Eg - June 2025
- `Year` — year only

---

## Date Adjusters

```java
// Built-in adjusters
LocalDate firstTuesday = LocalDate.of(year, month, 1)
    .with(TemporalAdjusters.nextOrSame(DayOfWeek.TUESDAY));
```

Available `TemporalAdjusters` factory methods:
- `next(dayOfWeek)` / `nextOrSame(dayOfWeek)`
- `previous(dayOfWeek)` / `previousOrSame(dayOfWeek)`
- `dayOfWeekInMonth(n, dayOfWeek)` — nth weekday of the month
- `lastInMonth(dayOfWeek)`
- `firstDayOfMonth()` / `lastDayOfMonth()`
- `firstDayOfNextMonth()`
- `firstDayOfYear()` / `lastDayOfYear()`
- `firstDayOfNextYear()`

### Custom Adjusters

```java
// Preferred: use ofDateAdjuster to avoid manual cast
TemporalAdjuster NEXT_WORKDAY = TemporalAdjusters.ofDateAdjuster(w -> {
    LocalDate result = w;
    do {
        result = result.plusDays(1);
    } while (result.getDayOfWeek().getValue() >= 6);
    return result;
});

LocalDate backToWork = today.with(NEXT_WORKDAY);
```

---

## Local Time

```java
LocalTime now = LocalTime.now();
LocalTime t = LocalTime.of(22, 30);
LocalTime t = LocalTime.of(22, 30, 0, 0);  // hour, minute, second, nano

t.plusHours(8)
t.minusMinutes(15)
t.withHour(12)
t.withMinute(0)

t.getHour()    // 0–23
t.getMinute()  // 0–59
t.getSecond()  // 0–59
t.getNano()    // 0–999,999,999

t.toSecondOfDay()
t.toNanoOfDay()

t.isBefore(other)
t.isAfter(other)
```

- `plus`/`minus` operations wrap around a 24-hour day
- AM/PM is a formatting concern, not part of `LocalTime`
- `LocalDateTime` — combines date and time; suitable for fixed time zones but not DST calculations

---

## Zoned Time

### Key Concepts

- Time zones managed by IANA database (https://www.iana.org/time-zones); updated several times per year
- Zone IDs: `America/New_York`, `Europe/Berlin`, `UTC`
- `ZoneId.getAvailableZoneIds()` — ~600 IDs
- __UTC__ — Coordinated Universal Time; no daylight savings

```java
ZoneId zone = ZoneId.of("America/New_York");
ZonedDateTime zdt = ZonedDateTime.of(1969, 7, 16, 9, 32, 0, 0, zone);
ZonedDateTime zdt = localDateTime.atZone(zone);
ZonedDateTime zdt = ZonedDateTime.of(localDate, localTime, zone);

Instant i = zdt.toInstant();
ZonedDateTime zdt = instant.atZone(zone);

zdt.withZoneSameInstant(otherZone)  // same moment, different zone display
zdt.withZoneSameLocal(otherZone)    // same local time, different zone
```

### DST Complications

- Clocks spring forward: constructing a time in the skipped hour yields the post-skip time
    - Eg - constructing `2013-03-31 02:30 Europe/Berlin` yields `03:30`
- Clocks fall back: ambiguous times resolve to the earlier of the two instants
    - Adding 1 hour to the ambiguous time keeps the same clock time but changes the UTC offset

### Arithmetic Across DST Boundaries

```java
// WRONG: adds 7 * 86400 seconds; meeting shifts by 1 hour across DST boundary
ZonedDateTime next = meeting.plus(Duration.ofDays(7));

// CORRECT: advances by 7 calendar days
ZonedDateTime next = meeting.plus(Period.ofDays(7));
```

- `OffsetDateTime` — time with UTC offset but no zone rules; for network protocols, not human time

---

## Formatting and Parsing

### Predefined Formatters

```java
DateTimeFormatter.ISO_LOCAL_DATE          // 1969-07-16
DateTimeFormatter.ISO_LOCAL_TIME          // 09:32:00
DateTimeFormatter.ISO_LOCAL_DATE_TIME     // 1969-07-16T09:32:00
DateTimeFormatter.ISO_OFFSET_DATE_TIME    // 1969-07-16T09:32:00-04:00
DateTimeFormatter.ISO_ZONED_DATE_TIME     // 1969-07-16T09:32:00-04:00[America/New_York]
DateTimeFormatter.ISO_INSTANT             // 1969-07-16T13:32:00Z
DateTimeFormatter.RFC_1123_DATE_TIME      // Wed, 16 Jul 1969 09:32:00 -0400
```

### Locale-Specific Formatters

```java
DateTimeFormatter f = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.LONG);
String s = f.format(apollo11launch);                          // July 16, 1969 9:32:00 AM EDT
String s = f.withLocale(Locale.FRENCH).format(apollo11launch); // 16 juillet 1969 09:32:00 EDT
```

Styles: `SHORT`, `MEDIUM`, `LONG`, `FULL`

### Locale-Adapted Pattern

```java
DateTimeFormatter f = DateTimeFormatter.ofLocalizedPattern("yMMMd");
f.withLocale(Locale.US).format(date)      // Jul 16, 1969
f.withLocale(Locale.GERMANY).format(date) // 16. Juli 1969
f.withLocale(Locale.CHINA).format(date)   // 1969年7月16日
```

- Specify only field symbols in descending size; no separators — locale provides them
- Use `j` for locale-appropriate 12/24-hour format

### Custom Pattern

```java
DateTimeFormatter f = DateTimeFormatter.ofPattern("E yyyy-MM-dd HH:mm");
// Wed 1969-07-16 09:32
```

Key pattern letters (case-sensitive):

| Symbol | Meaning | Example |
|---|---|---|
| `G` | Era | `AD` / `Anno Domini` |
| `y` | Year | `yy`=69, `yyyy`=1969 |
| `Y` | Week-numbering year | differs from `y` near year-end |
| `M` | Month | `M`=7, `MM`=07, `MMM`=Jul, `MMMM`=July |
| `d` | Day of month | `d`=6, `dd`=06 |
| `D` | Day of year | |
| `E` | Day of week | `E`=Wed, `EEEE`=Wednesday |
| `H` | Hour (0–23) | `H`=9, `HH`=09 |
| `h` | Hour (1–12) | |
| `a` | AM/PM | `AM` |
| `m` | Minute | `mm`=02 |
| `s` | Second | `ss`=00 |
| `S` | Fraction of second | |
| `n` | Nanosecond | |
| `VV` | Zone ID | `America/New_York` |
| `z` | Zone name | `EDT` / `Eastern Daylight Time` |
| `x`/`X` | Zone offset | `x`=-04, `xx`=-0400, `xxx`=-04:00; `X` uses Z for UTC |

> [!NOTE]
> `M` = month, `m` = minute. `d` = day of month, `D` = day of year. `s` = second, `S` = fraction of second. `y` = year, `Y` = week-numbering year (dangerous near Jan 1).

### Parsing

```java
LocalDate d = LocalDate.parse("1903-06-14");  // uses ISO_LOCAL_DATE
ZonedDateTime zdt = ZonedDateTime.parse(
    "1969-07-16 03:32:00-0400",
    DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ssxx"));
```

### DayOfWeek / Month Display Names

```java
for (DayOfWeek w : DayOfWeek.values())
    w.getDisplayName(TextStyle.SHORT, Locale.ENGLISH);  // Mon Tue ...
```

---

## Interoperating with Legacy Code

| Conversion | To Legacy | From Legacy |
|---|---|---|
| `Instant` ↔ `java.util.Date` | `Date.from(instant)` | `date.toInstant()` |
| `ZonedDateTime` ↔ `GregorianCalendar` | `GregorianCalendar.from(zdt)` | `cal.toZonedDateTime()` |
| `Instant` ↔ `java.sql.Timestamp` | `Timestamp.from(instant)` | `timestamp.toInstant()` |
| `LocalDateTime` ↔ `java.sql.Timestamp` | `Timestamp.valueOf(ldt)` | `timestamp.toLocalDateTime()` |
| `LocalDate` ↔ `java.sql.Date` | `Date.valueOf(ld)` | `date.toLocalDate()` |
| `LocalTime` ↔ `java.sql.Time` | `Time.valueOf(lt)` | `time.toLocalTime()` |
| `DateTimeFormatter` → `java.text.DateFormat` | `formatter.toFormat()` | — |
| `ZoneId` ↔ `java.util.TimeZone` | `TimeZone.getTimeZone(id)` | `timeZone.toZoneId()` |
| `Instant` ↔ `FileTime` | `FileTime.from(instant)` | `fileTime.toInstant()` |