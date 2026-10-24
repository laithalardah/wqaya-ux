# Wiqaya (وقاية): the full plan

**One line:** the allergy goes wherever the child goes. Anyone who feeds the child (grandparents, drivers, school,
other parents) scans a product. Gemma 4 reads the label in any language, the phone checks it against the child's
profile, and the family sees the result. Nobody, not us and not the AI, ever learns who is allergic to what.

Hack Day Amman (JOSA × MLH), theme **Digital Health & Wellness**, challenge **Best Open Source AI Project for
Social Good**. Planning only: nothing is built or committed until the team says go.

---

## 1. The problem (facts checked)
- **Food allergy:** 11.5% of 1,629 Jordanian schoolchildren had a parent-reported food allergy, and only 5.7% had a
  formal diagnosis. Most common: eggs, cow's milk, peanuts (2025 study).
- **Celiac disease:** about 1 in 124 schoolchildren in Irbid on a blood screen, and underdiagnosed.
- **Why it's hard in Jordan:**
  - Much packaged food is imported, with the original label in another language and often an Arabic sticker.
  - Hidden names: malt extract (gluten), casein and whey (milk), tahini (sesame).
  - The people buying the food are often not the parents.
- **Regulation:** Arabic labels or stickers are mandatory and border checks are strict. "May contain" warnings
  are usually voluntary. Pitch the gaps the rules don't cover; never say "regulation fails".

## 2. Design rules
1. **Code owns every verdict.** Gemma reads, pairs, suspects and writes; it can only raise caution, never lower it.
2. **Three results:**
   - **green** only when the full ingredient list was read and every item was understood;
   - **amber** for "couldn't verify";
   - **red** when an avoided ingredient is found.
   Unknown is never shown as safe. **Zero false greens** is the number we defend on stage.
3. **No medical advice.** It quotes the label. The emergency card shows the plan *the child's doctor* wrote. Every
   result says "a check, not a guarantee".
4. **Privacy by design** (section 5). Health data never leaves the family's phones unencrypted, and Gemma never
   sees it.

## 3. Features

### P0: must work in the demo
| # | Feature | Done when |
|---|---|---|
| F1 | **Family profiles:** avoided allergens (the EU's 14 major allergens plus custom ones), celiac, halal, vegetarian, and a "may contain" policy | A parent creates 2 profiles in under a minute |
| F2 | **Barcode scan:** camera reads the code (zxing) and looks it up in Open Food Facts | A known product gives a verdict at once |
| F3 | **Label photo:** Gemma 4 reads the ingredients, "contains" and "may contain" in any language into structured data, and says whether the list is complete and legible | Turkish, Italian, English and Arabic labels read correctly |
| F4 | **Allergen check, on the phone:** matching against Open Food Facts' allergen lists, the Jordanian pack and E-numbers; red, amber or green for each member, with the label words quoted | Benchmark: every allergen caught, 0 false greens |
| F5 | **Jordanian hidden-ingredients pack**, published as open data | See section 7 |
| F6 | **Original label vs Arabic sticker** (when a product has one): Gemma pairs the lists across languages; code flags anything in the original that's missing from the sticker | A missing "may contain nuts" line is caught |
| F7 | **Plain explanation** in Arabic or English, read aloud on request | Facts only |
| F8 | **Family circle, ask and approve:** a caregiver joins by QR code; an amber result → "ask Sara's mum" → the parent sees the product, the photo and the flagged words → approve or refuse → the answer is back in seconds | Works across two phones on stage |
| F9 | **Emergency button:** one tap calls 911; a card in Arabic and English with the doctor's action plan, allergies and medicines; an SMS to the family with the location and what was eaten | Works on any phone |

### P1: if time allows
| # | Feature |
|---|---|
| F10 | **Safe list:** approved products are green for the whole family, tied to that recipe |
| F11 | **Recipe-change alert:** a safe-list product with a changed ingredient list turns amber, with the difference shown |
| F12 | **Unknown ingredients:** Gemma suggests a likely allergen source (amber, "Gemma thinks"), e.g. lecithin with no source |
| F13 | **"Tell the cook" card:** the child's allergies with every hidden name, in Arabic and English |
| F14 | **Epinephrine pen expiry:** read the date from the box, remind the family 30 days before |
| F15 | **Child mode:** a big green, amber or red face, with a spoken result |

### P2: say these are "next steps" in the pitch
- Class and party mode.
- Reporting a bad sticker to the JFDA.
- Recall watch.
- Fine-tuning Gemma for label reading.
- Contributing missing products to Open Food Facts.

## 4. How it's built

### Pieces
```
 Phone: installable web app (HTTPS)          Our server (open source, no user data)
 ┌──────────────────────────────────┐        ┌────────────────────────────────────────┐
 │ camera + zxing barcode           │ label  │ /read   → Gemma 4 (label image/text only)│
 │ ALLERGEN ENGINE (runs here)      │ ─────▶ │ /product→ Open Food Facts lookup/cache  │
 │ profiles, scan history (encrypted│ ◀───── │ /relay  → encrypted family messages     │
 │   on the phone)                  │ facts  │ holds the Gemma API key (never in app)  │
 │ emergency: tel:911, sms:         │        └───────────────┬────────────────────────┘
 │ family circle (WebCrypto)        │                        ▼
 └──────────────────────────────────┘           Gemma 4 (Gemini API, paid tier,
                                                 or our own Gemma server)
```
- **Phone app:** React and Vite, installable, hosted on HTTPS (for example Vercel). HTTPS also lets the phone use
  the camera.
- **Server:** Python FastAPI on a small cloud host, with three endpoints: `/read` for Gemma, `/product` for Open Food
  Facts with a cache, and `/relay` for the encrypted family messages.
- **Gemma 4:** `gemma-4-26b-a4b-it` (or `gemma-4-31b-it`) through the Gemini API, which takes images.
  - **Paid tier only:** on the free tier Google may use requests to improve its products and human reviewers may
    read them; on the paid tier it doesn't.
  - **Alternative:** run Gemma 4 ourselves (open weights, e.g. Ollama on our own server), so no third party sees
    even the label photos.
- **The allergen engine runs on the phone** (TypeScript, with the Jordanian pack and allergen lists shipped as
  JSON). This is the key privacy decision: the server and Gemma return facts about the *product*, and only the
  phone knows the *people*.

### Gemma 4: what it does
| Call | Input (product data only) | Output | Feature |
|---|---|---|---|
| **G1 read_label** (vision) | the cropped label photo | `{language, product_name, ingredients[], contains, may_contain, list_complete, legible}` | F3 |
| **G2 pair_lists** | original list + Arabic list | `[{original_i, arabic_i or null}]` | F6; code computes the gaps |
| **G3 suspect_unknown** | unrecognised ingredient strings | `[{text, likely_allergens[], reason}]` | F12; amber only |
| **G4 explain** | **every** allergen found in the product (not only the child's) | short facts in Arabic or English | F7; the phone shows the parts that matter for each member |
| **Scan agent** (tool calling) | the scan request | Gemma calls tools: `lookup_barcode`, `read_label`, `compare_labels`; asks for a retake when the list is cut off | the photo path; **the verdict is still computed on the phone** |

- **Structured output:** request JSON; if the Gemma API endpoint doesn't support a JSON schema or tool calling for
  Gemma, ask for JSON in the prompt and validate it in code. Check this in the first hour.
- **The card texts** (F13 and the emergency card) are built **on the phone from templates.** No health data goes to
  Gemma.

### How a verdict is decided (on the phone, in code only)
For each member, and each allergen they avoid:
- **red:** a match in the ingredients or the "contains" line, or a "may contain" match when the policy is
  "avoid". Also red if a sticker gap hides an avoided allergen.
- **amber:**
  - a "may contain" match when the policy is "allow";
  - the list is incomplete or unreadable;
  - the product is unknown;
  - an ingredient isn't recognised;
  - Gemma suspects a source;
  - the recipe changed since approval;
  - a sticker gap without an allergen.
- **green:** only if the list is complete and legible, every ingredient is recognised, nothing is avoided, and
  both labels were compared when there are two. Or the product is on the safe list with the same recipe.

### Data model
- **On the phone, encrypted:**
  - Member: `name, avoid[], diets[], traces_policy, emergency{plan, meds, epi_expiry, contacts}`;
  - Scan: `product, scanned_by, time, verdict per member, evidence`;
  - SafeEntry.
- **On the server (not personal):** a product cache, and encrypted circle messages that expire.

## 5. Privacy design (the 5% criterion, and the reason families will trust it)

### What each party can see
| Data | Phone | Our server | Gemma (API) |
|---|---|---|---|
| Names, allergies, conditions, medicines, doctor's plan | ✓ (encrypted) | ✗ (only encrypted blobs it can't read) | ✗ never |
| Who is allergic to what | ✓ | ✗ | ✗ |
| Label photo | ✓ | passes through, not stored | ✓ (cropped label only) |
| Barcode scanned | ✓ | ✓ (not linked to a person) | ✗ |
| Location | only in the emergency SMS, composed on the phone | ✗ | ✗ |
| Scan history | ✓ and family circle (encrypted) | ✗ | ✗ |

### Rules
1. **Gemma gets the product, never the person.** It receives a cropped label image or ingredient text. Profiles,
   names, ages and conditions are never in a prompt. Explanations cover *every* allergen found in the product, so
   even the request doesn't reveal which allergy the child has.
2. **Clean images.** The phone crops to the label, re-encodes the image from pixels (dropping EXIF, GPS and device
   data) and sends only that.
3. **No accounts.** No email, no phone number, no login. A family is a random circle ID plus a key inside a QR code.
4. **End-to-end encryption:**
   - profiles and messages are encrypted on the phone with AES-GCM, using the family key from the QR code;
   - profile edits are signed by a parent's key (ECDSA), so only parents can change a profile;
   - the server stores only encrypted data, which expires after 30 days.
5. **Nothing kept on the server:** label images are deleted right after Gemma replies; logs record no images, no
   text and no IP addresses.
6. **AI provider terms:**
   - **Gemini API paid tier only:** prompts aren't used to improve Google's products. They may be cached briefly in
     any country where Google has facilities.
   - **Option:** run Gemma ourselves (open weights).
   - **Never** the free tier or AI Studio for real users.
7. **The family stays in control:** "delete everything" on each phone; export the profile; a parent can remove a
   caregiver, which creates a new circle key.
8. **Children's health data** is a sensitive category under Jordan's Personal Data Protection Law (2023), so:
   - collect the minimum;
   - get a parent's consent at setup;
   - write a plain privacy note in Arabic and English.
9. **Secrets:** the Gemma API key lives only in the server's environment; never in the app or the repository.
10. **Open source:** anyone can check that all of the above is true.

## 6. Data sources and licences
| Source | Use | Note |
|---|---|---|
| Open Food Facts database | barcode lookup, cached on the server | ODbL; limits of 15 reads and 10 searches per minute, so cache and prefetch products sold in Jordan |
| Open Food Facts allergen and ingredient lists | synonyms | Confirm the licence of the taxonomy files |
| Jordanian hidden-ingredients pack | ours | Publish under ODbL or CC-BY |
| Gemma 4 | model | Gemma terms of use; credit it |
| Our code | — | Apache-2.0 |

## 7. Jordanian pack (first version, check each entry with a second source)
- **Sesame:** tahini (طحينة), halawa (حلاوة), za'atar mix (زعتر), ka'ak topping (كعك).
- **Milk:** samneh/ghee (سمنة), jameed (جميد, in mansaf), labneh (لبنة), kishk (كشك, also wheat).
- **Gluten:** freekeh (فريكة), bulgur (برغل), semolina (سميد), malt (شعير/مالت).
- **Tree nuts:** pine nuts (صنوبر), as an explicit profile option, because not every regulator classes them as tree
  nuts.
- **E-numbers:**
  - E322 lecithin: amber unless the source is stated;
  - E471: amber for halal;
  - E441 gelatin: halal and vegetarian check;
  - E120 carmine: from insects (vegetarian; halal views differ, so amber);
  - E904 shellac: vegetarian check.

## 8. Benchmark
- **30 labels:**
  - 10 real products;
  - 20 synthetic labels with planted allergens, hidden names, sticker lines left out, blurry or cut-off lists, and
    a changed recipe.
- **Metrics:** allergens caught (target: all), **false greens (must be 0)**, false alarms, and **Gemma against plain
  OCR** for ingredient extraction.
- **Privacy test:** capture every request the server sends to Gemma during the benchmark and show that none contains
  a name, an allergy profile or a location.
- **Tests:** unit tests for normalisation, matching, the verdict logic, gap detection and encryption round-trips.

## 9. Build plan (team of 4)
| Block | Person 1: Gemma | Person 2: rules and data | Person 3: app | Person 4: family, privacy, pitch |
|---|---|---|---|---|
| Hour 0–1 | Gemma API: image input, JSON, tool calling | Open Food Facts cache, allergen lists | App skeleton, HTTPS hosting, camera | Buy products; encryption design |
| 1–4 | G1, G2, scan agent with fallback | Allergen engine on the phone, Jordanian pack, verdict | Profiles, scan, result screens | Relay, invite QR, encryption, signing |
| 4–6 | G3, G4 | Benchmark set and runner, privacy test | Ask and approve, emergency card | Emergency SMS, safe list, privacy note |
| 6–7 | Prompt tuning | Run the benchmark, record numbers | Polish, Arabic and English | Pitch slides, demo script |
| Last hour | **All:** rehearse twice, freeze the code | | | |

## 10. Demo (3 minutes)
1. **Hook:** "11.5% of Jordanian schoolchildren have a parent-reported food allergy, and the person buying their
   snacks is often their grandmother."
2. **Grandma's phone:** an imported product comes back amber. The original says "may contain nuts" in Italian; the
   Arabic sticker doesn't. (Without a real sticker product, use a synthetic label and say so.)
3. **"Ask Sara's mum":** the parent's phone, the evidence, refuse. Grandma's screen turns red.
4. **Hummus crackers:** tahini caught as sesame.
5. **Emergency:** the card plus the SMS "Sara ate X at 4:12 pm".
6. **Privacy in one slide:** "Gemma sees the label, never the child." Show a captured Gemma request: just a label.
7. **Numbers:** all allergens caught, 0 false greens, Gemma against OCR; open source.

## 11. Scoring against the criteria
| Criterion | Weight | Evidence |
|---|---|---|
| Challenge alignment | 10% | "Stop problems before they start", "smart open source health tools" |
| Problem & impact | 15% | The Jordanian numbers; caregivers; imported labels |
| Functionality & practicality | 20% | Live scans of real products, two phones, emergency |
| Technical quality & execution | 15% | Rules-first verdicts, benchmark, tests, fallback paths |
| Innovation & creativity | 20% | Sticker comparison, ask and approve, the Jordanian pack, recipe-change alert |
| Open source AI integration | 15% | Gemma 4 (vision, tool calling); Open Food Facts; open code and data |
| Privacy & responsible design | 5% | Gemma never sees the person; end-to-end encryption; no accounts; zero false greens |

## 12. Risks and open checks
- **Gemma via the API:** confirm image input, JSON output and tool calling for the Gemma 4 model IDs. If tool
  calling isn't supported there, the fixed pipeline G1 → check → G2 → G4 does the same job.
- **Open Food Facts coverage** of products sold in Jordan: if low, the photo path carries the demo.
- **Facts to double-check:** pine nut classification, E120 halal status, the taxonomy licence.
