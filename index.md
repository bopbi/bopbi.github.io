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

### [Android AOP](https://flyjingfish.github.io/AndroidAOP/)
###### Date: 2025/01/03

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
