# Bitmovin Player UI
The Bitmovin Adaptive Streaming Player UI

Read more about the usage, as well as other important information on Bitmovin's Adaptive Streaming Player itself at https://bitmovin.com/ and https://bitmovin.com/player-documentation/.

## Getting Started

 1. Install node.js
 2. Install Gulp: `npm install --global gulp-cli`
 3. Install required npm packages: `npm install`
 4. Run Gulp tasks (`gulp --tasks`)
  * `gulp` to build project into `dist` directory
  * `gulp watch` to develop and rebuild changed files automatically
  * `gulp serve` to open test page in browser, build and reload changed files automatically
  * `gulp lint` to lint TypeScript and SASS files
  * `gulp build-prod` to build project with minified files into `dist` directory
  
To just take a look at the project, also run `gulp serve`.

## Introduction

This repository contains the Bitmovin Player UI framework introduced with the 7.0 release. 
It is designed as a flexible and modularized layer on the player API that replaces the old integrated monolithic UI, enabling customers and users of the player to easily customize the UI to their needs in design, structure, and functionality. It makes it extremely easy and straightforward to add additional control components and we encourage our users to proactively contribute to our codebase.

The framework basically consists of a `UIManager` that handles initialization and destruction of the UI, and components extending the `Component` base class. Components provide specific functionality (e.g. `PlaybackToggleButton`, `ControlBar`, `SeekBar`, `SubtitleOverlay`) and usually consist of two files, a TypeScript `.ts` file containing control code and API interaction with the player, and a SASS `.scss` file containing the visual style.

A UI is defined by a tree of components, making up the UI *structure*, and their visuals styles, making up the UI *skin*. The root of the structure always starts with a `UIContainer` (or a subclass, e.g. `CastUIContainer`), which is a subclass of `Container` and can contain other components, like any other components extending this class (usually layout components, e.g. `ControlBar`). Components that do not extend the `Container` cannot contain other components and therefore make up the leaves of the UI tree.

## Customizing the UI

There are basically two approaches to customize the UI. The simple approach is to go with the built-in UI of the player and adjust the styling to your liking with CSS. the advanced approach is to replace the built-in UI with a your own build from this repository.

### Styling the built-in UI

When using the built-in UI, you can style it to your linking with CSS by overwriting our default styles, as documented in our [CSS Class Reference](https://bitmovin.com/player-documentation/css-class-reference/).

### Replacing the built-in UI

To use the player with a custom UI, you need to deactivate the built-in UI, include the necessary `js` and `css` files into your HTML and create and attach your own
UI instance with the `UIManager`.

 * Deactivate the built-in UI by setting `ux: false` in the `style` config of the player ([https://bitmovin.com/player-documentation/player-configuration/](Player Configuration Guide))
 * Build the UI framework (e.g. `gulp build-prod`) and include `bitmovinplayer-ui.min.js` and `bitmovinplayer-ui.min.css` (or their non-minified counterparts) from the `dist` directory
 * Create your own UI instance with the `UIManager.Factory` once the player is loaded

```js
var config = {
    ...,
    source: {
      ...
    },
    style: {
        ux: false // disable the built-in UI
    }
};

bitmovin.player('player-id').setup(config).then(function (player) {
  var myUiManager = bitmovin.playerui.UIManager.Factory.buildDefaultUI(player);
});
```

### Building a custom UI structure

Instead of using predefined UI structures from the `UIManager.Factory`, you can easily create a custom structure. For examples on how to create such UI structures, take a look at the `UIManager.Factory`.

A simple example on how to create a custom UI with our default skin that only contains a playback toggle overlay (an overlay with a large playback toggle button) looks as follows:

```js
// Definition of the UI structure
var mySimpleUi = new UIContainer({
  components: [
    new PlaybackToggleOverlay()
  ],
  cssClasses: ['ui-skin-modern']
});

bitmovin.player('player-id').setup(config).then(function (player) {
  // Add the UI to the player
  var myUiManager = new bitmovin.playerui.UIManager(player, mySimpleUi, null);
});
```

### UIManager

The `UIManager` manages UI instances and is used to add and remove UIs to/from the player. To add a UI to the player, construct a new instance and pass the `player` object, a UI structure (`UIContainer`), a second UI structure to be displayed during ads or `null`, and an optional configuration object. To remove a UI from the player, just call `release()` on your UIManager instance.

```js
// Add UI (e.g. at player initialization)
var myUiManager = new bitmovin.playerui.UIManager(player, mySimpleUI, null);

// Remove UI (e.g. at player destruction)
myUiManager.release();
```

UIs can be added and removed anytime during the player's lifecycle, which means UIs can be dynamically adjusted to the player, e.g. by listening to events. It is also perfectly possible to manage multiple UIs in parallel.

Here is an example on how to display a special UI in fullscreen mode:

```js
bitmovin.player('player-id').setup(config).then(function (player) {
  var myUiManager = new bitmovin.playerui.UIManager(player, myWindowUi, null);
  
  player.addEventHandler(player.EVENT.ON_FULLSCREEN_ENTER, function () {
    myUiManager.release();
    myUiManager = new bitmovin.playerui.UIManager(player, myFullscreenUi, null);
  });
  
  player.addEventHandler(player.EVENT.ON_FULLSCREEN_EXIT, function () {
    myUiManager.release();
    myUiManager = new bitmovin.playerui.UIManager(player, myWindowUi, null);
  });
});
```
#### Factory

`UIManager.Factory` provides a few predefined UI structures and styles, e.g.:

 * `buildDefaultUI`: The default UI as used by the player by default
 * `buildDefaultCastReceiverUI`: A light UI specifically for Google Cast receivers
 * `buildLegacyUI`: ported legacy UI style from player <= version 6

You can easily test and switch between these UIs in the UI playground.

### Components

For the list of available components check the `src/ts/components` directory. Each component extends the `Component` base class and adds its own configuration interface and functionality. Components that can container other components as child elements extend the `Container` component. Components are associated to their CSS styles by the `cssClass` config property (prefixed by the `cssPrefix` config property and the `$prefix` SCSS variable).

Custom components can be easily written by extending any existing component, depending on the required functionality.

#### Component Configuration

All components can be directly configured with an optional configuration object that is the first and only parameter of the constructor and defined by an interface. Each component is either accompanied by its own configuration interface (defined in the same `.ts` file and named with the suffix `Config`, e.g. `LabelConfig` for a `Label`), or inherits the configuration interface from its superclass.

There is currently no way to change these configuration values on an existing UI instance, thus they must be passed directly when creating a custom UI structure.

The following example creates a very basic UI structure with only two text labels:

```js
var myUi = new UIContainer({
  components: [
    new Label({ text: "A label" }),
    new Label({ text: "A hidden label", hidden: true })
  ],
  cssClasses: ['ui-skin-modern']
});
```

The `UIContainer` is configures with two options, the `components`, an array containing child components, and `cssClasses`, an array with CSS classes to be set on the container. The labels are configures with some `text`, and one label is initially hidden by setting the `hidden` option.

### UI Configuration

The `UIManager` takes an optional global configuration object that can be used to configure certain content on the UI.

```js
var myUiConfig = {
  metadata: {
    title: 'Video title',
    description: 'Video description...'
  },
  recommendations: [
    {title: 'Recommendation 1: The best video ever', url: 'http://bitmovin.com', thumbnail: 'http://placehold.it/300x300', duration: 10.4},
    {title: 'Recommendation 2: The second best video', url: 'http://bitmovin.com', thumbnail: 'http://placehold.it/300x300', duration: 64},
    {title: 'Recommendation 3: The third best video of all time', url: 'http://bitmovin.com', thumbnail: 'http://placehold.it/300x300', duration: 195}
  ]
};

var myUiManager = new bitmovin.playerui.UIManager(player, myUi, null, myUiConfig);
```

All of the configuration properties are optional. If `metadata` is set, it overwrites the metadata of the player configuration. If `recommendations` is set, a list of recommendations is shown in the `RecommendationOverlay` at the end of playback. For this to work, the UI must contain a `RecommendationOverlay`, like the default player UI does.

### UI Playground

The UI playground can be launched with `gulp serve` and opens a page in a local browser window. On this page, you can switch between different sources and UI styles, trigger API actions and observe events.

This page uses BrowserSync to sync the state across multiple tabs and browsers and recompiles and reloads automatically files automatically when any `.scss` or `.ts` files are modified. It makes a helpful tool for developing and testing the UI.
