# Core Java Vol. II — Chapter 7: Internationalization

## Locales

### Locale Components

- __Language__ — 2–3 lowercase letters (ISO 639-1): `en`, `de`, `zh`
- __Script__ (optional) — 4 letters, initial uppercase: `Latn`, `Cyrl`, `Hant`
- __Country__ (optional) — 2 uppercase letters (ISO 3166-1): `US`, `CH`, `DE`
- __Variant__ (optional) — miscellaneous features; rarely used today
- __Extension__ (optional) — local preferences; starts with `u-` (Unicode) or `x-` (arbitrary)
    - Eg - `u-nu-thai` — Thai numerals
    - Eg - `u-ca-japanese` — Japanese calendar

### Creating Locales

```java
Locale usEn = Locale.forLanguageTag("en-US");     // from tag string
Locale german = Locale.of("de");                   // language only
Locale swissGerman = Locale.of("de", "CH");        // language + country

Locale.GERMANY       // predefined country locales
Locale.GERMAN        // predefined language-only locales
Locale.ROOT          // language-neutral; for machine-processed strings

Locale.getDefault()                            // system locale
Locale.getDefault(Locale.Category.DISPLAY)     // locale for display messages
Locale.getDefault(Locale.Category.FORMAT)      // locale for number/date formatting

locale.toLanguageTag()   // "de-CH"
locale.getDisplayName()  // "German (Switzerland)" in current locale
locale.getDisplayName(Locale.GERMAN)  // "Deutsch (Schweiz)"
locale.getLanguage()     // "de"
locale.getCountry()      // "CH"

Locale.getAvailableLocales()    // array
Locale.availableLocales()       // stream
Locale.getISOLanguages()
Locale.getISOCountries()
```

- Override at launch: `java -Duser.language=de -Duser.region=CH MyProgram`
- `toLowerCase(locale)` and `toUpperCase(locale)` — always pass locale; Turkish `I` is a known pitfall

---

## Number Formats

### `NumberFormat` Factory Methods

```java
NumberFormat nf  = NumberFormat.getNumberInstance(locale);
NumberFormat cf  = NumberFormat.getCurrencyInstance(locale);
NumberFormat pf  = NumberFormat.getPercentInstance(locale);
NumberFormat cnf = NumberFormat.getCompactNumberInstance(locale, NumberFormat.Style.SHORT);
// Style.SHORT → "123K"; Style.LONG → "123 thousand"

String s = nf.format(123456.78);
Number n = nf.parse("123.456,78");   // returns Double or Long; no leading whitespace
double x = n.doubleValue();
```

- `parse` is strict about leading whitespace — call `strip()` first
- Trailing non-numeric characters after the number are silently ignored
- Actual returned type is a subclass of `NumberFormat`; do not downcast without checking

### Configuring Formatters

```java
nf.setGroupingUsed(true);
nf.setMinimumFractionDigits(2);
nf.setMaximumFractionDigits(4);
nf.setMinimumIntegerDigits(1);
nf.setParseIntegerOnly(false);
```

### `DecimalFormat` with Custom Patterns

```java
var formatter = new DecimalFormat("0.00;(#)");
// positive: at least 1 integer digit, 2 fraction digits
// negative: accounting format in parentheses
```

Pattern syntax:
- `0` — required digit
- `#` — optional digit
- `,` — grouping separator position
- `.` — decimal separator
- `;` — separates positive and negative sub-patterns
- `¤` (U+00A4) — currency symbol placeholder
- `%` — multiply by 100 and append percent
- `‰` (U+2030) — multiply by 1000 (per mille)
- Single-quote prefix for literal special chars: `'#'` for literal `#`

### `DecimalFormatSymbols` Customisation

```java
DecimalFormatSymbols symbols = new DecimalFormatSymbols(locale);
symbols.setGroupingSeparator(',');
symbols.setDecimalSeparator('.');
((DecimalFormat) formatter).setDecimalFormatSymbols(symbols);
```

### Currencies

```java
NumberFormat euroFmt = NumberFormat.getCurrencyInstance(Locale.US);
euroFmt.setCurrency(Currency.getInstance("EUR"));  // show EUR in US format

Currency.getInstance("USD")        // by ISO 4217 code
Currency.getInstance(Locale.JAPAN) // by locale
currency.getCurrencyCode()         // "EUR"
currency.getSymbol()               // "$" or "US$" depending on locale
currency.getDefaultFractionDigits()
Currency.getAvailableCurrencies()
```

Common ISO 4217 codes: `USD`, `EUR`, `GBP`, `JPY`, `CNY`, `INR`, `RUB`

---

## Date and Time

```java
FormatStyle style = FormatStyle.LONG;  // SHORT, MEDIUM, LONG, FULL
DateTimeFormatter df = DateTimeFormatter.ofLocalizedDate(style).withLocale(locale);
DateTimeFormatter tf = DateTimeFormatter.ofLocalizedTime(style).withLocale(locale);
DateTimeFormatter dtf = DateTimeFormatter.ofLocalizedDateTime(style).withLocale(locale);

String s = df.format(LocalDate.now());
LocalDate d = LocalDate.parse(input, df);
```

- `LONG` and `FULL` styles for time only work meaningfully with `ZonedDateTime`
- Parsers reject non-existent dates by adjusting to last valid day of month
- Not suitable for raw user input without preprocessing

### Month and Day Names

```java
for (Month m : Month.values())
    System.out.println(m.getDisplayName(TextStyle.FULL, locale));

for (DayOfWeek w : DayOfWeek.values())
    System.out.println(w.getDisplayName(TextStyle.SHORT, Locale.ENGLISH));

DayOfWeek first = WeekFields.of(locale).getFirstDayOfWeek();
```

TextStyle values: `FULL`, `SHORT`, `NARROW` (each with a `_STANDALONE` variant)
- `STANDALONE` — for display outside a formatted date (matters in Finnish, etc.)

---

## Collation and Normalization

### `Collator`

```java
Collator coll = Collator.getInstance(locale);
words.sort(coll);  // Collator implements Comparator<Object>

coll.compare("Ångström", "Angstrom")  // negative, zero, or positive
coll.equals("Able", "able")           // depends on strength
```

### Strength Levels

```java
coll.setStrength(Collator.PRIMARY);    // only primary differences (A vs Z)
coll.setStrength(Collator.SECONDARY);  // + secondary differences (A vs Å)
coll.setStrength(Collator.TERTIARY);   // + tertiary differences (A vs a); default
coll.setStrength(Collator.IDENTICAL);  // no differences allowed
```

### Decomposition Modes

```java
coll.setDecomposition(Collator.NO_DECOMPOSITION);        // fastest; may miss equivalences
coll.setDecomposition(Collator.CANONICAL_DECOMPOSITION); // default; handles accents
coll.setDecomposition(Collator.FULL_DECOMPOSITION);      // also handles ligatures (™ = TM)
```

### Collation Keys (for repeated comparisons)

```java
CollationKey aKey = coll.getCollationKey(a);
int result = aKey.compareTo(coll.getCollationKey(b));  // fast; avoids re-decomposition
```

### Unicode Normalization

```java
String normalized = Normalizer.normalize(name, Normalizer.Form.NFD);
// NFD: decompose accents (Å → A + ̊)
// NFC: decompose then recompose (recommended for transmission)
// NFKD: full decomposition including ligatures
// NFKC: full decomposition then recompose
```

- W3C recommends NFC for Internet data transfer

---

## Message Formatting

### `MessageFormat`

```java
String pattern = "On {2,date,long}, a {0} destroyed {1} houses and caused {3,number,currency} of damage.";
String msg = MessageFormat.format(pattern, "hurricane", 99, new GregorianCalendar(1999, 0, 1).getTime(), 10.0E8);
```

Placeholder format: `{index}`, `{index,type}`, or `{index,type,style}`

Types and styles:
- `number` → `integer`, `currency`, `percent`, or a `DecimalFormat` pattern
- `date` → `short`, `medium`, `long`, `full`, or a `SimpleDateFormat` pattern
- `time` → same styles as date
- `choice` → choice format string (see below)

For a locale-specific formatter:

```java
var formatter = new MessageFormat(pattern, locale);
String msg = formatter.format(new Object[] { arg0, arg1, arg2 });
```

- Literal braces: surround with single quotes — `'{'`
- Curly apostrophes preferred for text containing apostrophes: `{0}'s`
- Straight single quotes must be doubled: `''`

### Choice Formats

```java
"{1,choice,0#no houses|1#one house|2#{1} houses}"
```

Format: `lowerBound#formatString|lowerBound#formatString|...`
- `#` — boundary is inclusive (≥)
- `<` — boundary is exclusive (>)
- `≤` (U+2264) — synonym for `#`
- `∞` (U+221E) — negative infinity (for the first boundary)
- The format string can itself contain `{index}` placeholders that get re-formatted

Eg - full message with choice:
"On {2,date,long}, {0} destroyed {1,choice,0#no houses|1#one house|2#{1} houses} and caused {3,number,currency} of damage."

---

## Text Boundaries

```java
BreakIterator iter = BreakIterator.getSentenceInstance(locale);
// also: getCharacterInstance, getWordInstance, getLineInstance
iter.setText(s);
int start = iter.first();
int end = iter.next();
while (end != BreakIterator.DONE) {
    String segment = s.substring(start, end);
    start = end;
    end = iter.next();
}
```

- `getCharacterInstance` — grapheme clusters (visible characters)
- `getWordInstance` — words and word separators (caller must distinguish)
- `getSentenceInstance` — sentences
- `getLineInstance` — line-break positions
- Alternative for grapheme clusters: `s.split("\\b{g}")`

---

## Text Input and Output

### Text Files

```java
Charset cs = Charset.forName("Windows-1252");
var out = new PrintWriter(filename, cs);
```

- Use UTF-8 whenever possible
- `native.encoding` system property (Java 18+) gives the system encoding name
- Before Java 18: `Charset.defaultCharset()` gives the native encoding

### Line Endings

- `println` always uses the platform line separator
- `\n` inside strings is NOT converted — use `printf` with `%n` for platform-independent newlines:

```java
out.printf("Hello%nWorld%n");  // \r\n on Windows, \n elsewhere
```

### Console Encoding

```java
// Java 18+: stdout.encoding and stderr.encoding system properties give console encoding
// Switch Windows console to UTF-8:
// chcp 65001
// java -Dfile.encoding=UTF-8 MyProgram   (older Java versions)

// Use console reader for correct locale-aware input
Scanner sc = System.console() != null
    ? new Scanner(System.console().reader())
    : new Scanner(System.in);
```

### UTF-8 Byte Order Mark

- Java does NOT strip BOM (U+FEFF) from input — must strip manually
- `javac` fails if source file starts with a BOM

### Source File Encoding

```bash
javac -encoding UTF-8 MyProgram.java  # Java 18+: UTF-8 is the default
```

---

## Resource Bundles

### Naming Convention

MyProgramStrings.properties            # fallback

MyProgramStrings_de.properties         # language-only

MyProgramStrings_de_DE.properties      # language + country

MyProgramResources.java                # class-based bundle

MyProgramResources_de_DE.java

### Loading and Lookup

```java
ResourceBundle bundle = ResourceBundle.getBundle("MyProgramStrings", locale);
String label = bundle.getString("computeButton");
Object obj   = bundle.getObject("backgroundColor");
String[] arr = bundle.getStringArray("weekdays");
```

### Lookup Order

`getBundle` tries in order:
1. `baseName_currentLocaleLanguage_currentLocaleCountry`
2. `baseName_currentLocaleLanguage`
3. `baseName_defaultLocaleLanguage_defaultLocaleCountry`
4. `baseName_defaultLocaleLanguage`
5. `baseName`

- Parent bundles remain as fallback — missing keys bubble up the hierarchy
- Class bundles take priority over property files of the same base name

### Property Files

```properties
computeButton=Rechnen
colorName=black
defaultPaperSize=210×297
```

- Encoded as UTF-8 (Java 9+)
- Before Java 9: ASCII only; non-ASCII must use `\uXXXX` escapes

### Class-Based Bundles

```java
public class ProgramResources_de extends ListResourceBundle {
    private static final Object[][] contents = {
        { "backgroundColor", Color.black },
        { "defaultPaperSize", new double[] { 210, 297 } }
    };
    public Object[][] getContents() { return contents; }
}
```

- Extend `ListResourceBundle` for simple key-object pairs
- Extend `ResourceBundle` directly for custom lookup: implement `getKeys()` and `handleGetObject(key)`