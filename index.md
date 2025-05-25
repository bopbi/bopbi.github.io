# Bopbi's mind

I'm Bobby, an Android Dev and (still) an app hobbyist

## Recent TIL

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
