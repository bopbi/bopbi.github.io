# Bopbi's mind

I'm Bobby, an Android Dev and (still) an app hobbyist

## Current Project

- [QRGen](https://bobbyprabowo.com/qrgen) : QR Code Generator for Android

## Recent TIL

<!-- Post Template

### [Title](Url)
###### Date: 2025/08/24

Content here

---

-->

### [DroidKaigi Day 1](https://2025.droidkaigi.jp/en/timetable/9-11/)
###### Date: 2025/09/11

#### Building with AI in Kotlin
Talks is presented by Jetbrains Dev Rels

##### AI Assisted Coding in Kotlin on the IntelliJ Ultimate
Although the AI demo is mostly from a Video, its demoing Interesting stuff:
- Offline Code Assistance
- In-Editor Code Generation
- Agent Assisted Junie (EAP)
- Bytecode manipulation
- AI code generation using Project guideline

##### Building AI
- Existing MCP SDK from OpenAI, Anthropic
- Koog (AI Agent Framework) that can generate code on various platform (web / native) and can use major vendors LLM (somehow the promoted openrouter service, the openrouter UI remind me of ollama)
- first demo is a question and answer
- Message History, and Node (input trigger node type response, node can be chained)
- Koog can be used with any MCP (MCP can be found in mcpmarket)
- History compression to improve the resulted action since AI have issue if the context is too large

#### Composables beyond UI

##### Compose Concept
The Session begins with explaining how the Compose Work
- Composition Tree -> State Change -> Runtime will Recompose
- Compose Tree, ReusableComposeNode

##### Applying Compose to Audio Processing
Case [Koruri](https://github.com/Koruri/Koruri) Declarative Audio Processing Library
Wave manipulation (pitch, volume) drawed as a block
```
Chain {
  SineWave()
  Volume()
}

Mix {
  SineWave()
  SineWave()
}
```
the interesting is, the library is complete the Compose capability to making a multimedia (Synthesizer), since now the wave manipulation can integrate seamlessly in the same approach in composable structure and state (UI state like touch or slide)

#### Gen AI for Developers

- Cases of extracting info from a application UI (Chat App)
- Gemini nano capabilities, latest is nano V3
- Gemini nano improvement changes according to the device
- Dev need to Understand LLM "thinks" differently with human
- Preprocessing (Tokenization, Embedding) -> Decode (Auto Completer, Self Attention, Output Probablities)

Tips when make Gen AI (Prompt Handling)
- Start Small => a simple prompt can be something with multiple steps (workflow)
- Test and Test: Recommended Sample is 200 diverse item, metric to evaluate quality, Evaluate -> Refine Iterate
- Creative in Testing (like over the sample)
- Avoid hallucinations
- Emotional Stimuli
- Reframe your Prompt
- Common Practise (Proper Role / Persona, Premise Order Matter, add Delimiter, Write Prompt in English)
- Magic cue auto suggest
- start with d.android.com/ai kaggle.com/whitepaper-prompt-engineering

---

#### Android Librarian guide

- Care only public interface
- Minimize API Surface (internal + @JvmSynthetic, Kotlin explicitAPI)
- API Lifecycle by Deprecated Annotation (message, replaceWith, level), or Compatibility Table
- Tools to check Binary Complatibility issue
  - Binary Complatibility Validator Plugin by Jetbrains (generate .api files), product flavors
  - Metalava from Google
  - above can be hooks it on CI pipeline
- UI in Library
  - AAR file format
  - Minimizing Exposed Library => Transitive Resource
    - Public tag on the xml file (public.xml)
    - resourcePrefix in build.gradle
  - Dependency can go to the library dependency (Transitive Dependency)
  - Transitive R class
  - Transitive Class (Dex Limit), but non Transitive Class are enabled by default on Gradle 8.0

#### Server Driven UI

- Complex JSON that can parsed to generate UI
- Demo can reflect the json changes on the fly (font style, position, and complex UI)

---

### [Android Vitals Series](https://dev.to/pyricau/series/7827)
###### Date: 2025/08/24

Series of Article about Android Performance

---

### [Android AOP](https://flyjingfish.github.io/AndroidAOP/)
###### Date: 2025/08/03

These days, an app isn’t just about building features — it also needs proper logging (performance, analytics).
The problem is, teams often don’t have a clear agreement on how to do it, which leads to confusion when debugging and makes the business logic messy with logging code.

When i stumbled upon the AOP Concept it might the proper framework to help implementing the logs, lets see whether this one can help

to fix the java 11 requirement, need to install java 11 first, and add the line below on root `gradle.properties` then sync the project
```
org.gradle.java.home=/Library/Java/JavaVirtualMachines/temurin-11.jdk/Contents/Home
```

---

### [Android Makers: How to keep your app's secrets, secret](https://www.youtube.com/watch?v=H-wdOLCIiXA)
###### Date: 2025/06/26

Interesting tools (approach)
1. [Secrets Gradle Plugin for Android](https://developers.google.com/maps/documentation/android-sdk/secrets-gradle-plugin) using the same approach as the Google Maps SDK
2. [gitleaks](https://gitleaks.io/) a Static analytics tools
3. `git filter-branch` manually check git history
4. [git-filter-repo](https://github.com/newren/git-filter-repo) remove logged secret changes by rewrite git history (using python)
5. [BFG Repo-cleaner](https://rtyley.github.io/bfg-repo-cleaner/) remove logged secret changes by rewrite git history (using scala)
6. [hidden-secrets-gradle-plugin](https://github.com/klaxit/hidden-secrets-gradle-plugin) uses ndk with XOR (may be not maintained last commit are 2 years ago)
7. [bytemask](https://patilshreyas.github.io/bytemask/introduction.html) mask secrets that make it difficult to reverse engineering
8. [Sekret](https://github.com/DatL4g/Sekret) store secrets on NDK using KMM plugin, also provide secure logging by using for annotated compatible string/char property
9. Own API proxy servers

---

### [Jetpack Compose: Debugging recomposition](https://www.youtube.com/watch?v=SWBN0y0lFNY)
###### Date: 2025/05/25

Three Phase in Compose

1. Composition: What UI to show, building tree of composables
2. Layout: Where to Place UI, layout take those composables and works out where on the screen they will be shown
3. Drawing: How it renders, Draw everything to screen

Compose can skip a phase entirely if nothing has changed in it

Prefer lambda modifiers when using frequently changing state, the video show case is using the scroll provider lambda over y property when performing translation, and reading the state inside the graphicsLayer

---

### [Android Api Level](https://apilevels.com)
###### Date: 2025/01/03

Since i often forget to about the name of the Version, the codename
and the site is include the play store requirement changes, and major library info

---

### [How to Test Conflating Stateflow](https://zsmb.co/conflating-stateflows/)
###### Date: 2024/12/15

Separate assert and code triggering into 2 different set of `coroutine launch`
* the first one is for assertion, the launch should use `UnconfinedTestDispatcher`
* the other (last) one is for trigger the code / sequence / logic

Please note that when using flow the code triggering should still use a separate launch and not inside the `test` block

---
