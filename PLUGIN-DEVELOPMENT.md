# Origo Plugin Development Guide

This guide provides comprehensive documentation for creating plugins for origo-map. Whether you're building a simple UI control or a complex feature, this guide will help you get started and understand the plugin architecture.

## Table of Contents

1. [Introduction](#introduction)
2. [Getting Started](#getting-started)
3. [Plugin Architecture](#plugin-architecture)
4. [Creating Your First Plugin](#creating-your-first-plugin)
5. [Common Patterns](#common-patterns)
6. [Extension Points](#extension-points)
7. [API Reference](#api-reference)
8. [Localization Support](#localization-support)
9. [Building with Vite](#building-with-vite)
10. [Testing](#testing)
11. [Best Practices](#best-practices)
12. [Examples](#examples)

## Introduction

Origo plugins allow you to extend the core functionality of origo-map without modifying the core codebase. Plugins are ideal for:

- Features with external dependencies
- Specialized functionality not needed by average users
- Custom UI components and controls
- Integration with external services
- Domain-specific tools

### Plugin Types

Origo supports two types of plugins:

- **Controls**: UI components that provide interactive functionality (buttons, panels, tools)
- **Extensions**: Lightweight modifications or enhancements without UI elements

Most plugins are **Controls** as they typically provide user-facing features.

## Getting Started

### Prerequisites

- Node.js (LTS version or higher)
- Git
- Basic knowledge of JavaScript ES6+
- Familiarity with OpenLayers (the mapping library used by Origo)

### Quick Start

1. **Use the barebone plugin template** (recommended):
   ```bash
   git clone https://github.com/origo-map/barebone-plugin.git my-plugin
   cd my-plugin
   npm install
   ```

2. **Or start from scratch**:
   ```bash
   mkdir my-plugin
   cd my-plugin
   npm init -y
   ```

### Development Environment Setup

Install the required dependencies:

```bash
npm install --save-dev vite
```

For development, you'll need Origo's dependencies (these are typically provided by the host application):

```json
{
  "peerDependencies": {
    "ol": "^10.6.0"
  }
}
```

## Plugin Architecture

### Understanding Controls

Controls are the most common type of plugin. They:

1. Return a **Component** object (from Origo's UI framework)
2. Implement lifecycle hooks (`onInit`, `onAdd`, `onRender`)
3. Can access the viewer instance and all Origo APIs
4. Can create UI elements using Origo's UI components

### The Component System

Origo uses a lightweight component-based UI system. Every control must return a Component:

```javascript
import Origo from 'Origo';

const MyPlugin = function MyPlugin(options = {}) {
  let viewer;
  
  return Origo.ui.Component({
    name: 'myplugin',
    
    onInit() {
      // Called when component is created
      console.log('Plugin initialized');
    },
    
    onAdd(evt) {
      // Called when added to viewer
      viewer = evt.target;
      console.log('Plugin added to viewer');
    },
    
    onRender() {
      // Called when component should render
      console.log('Plugin rendered');
    }
  });
};

export default MyPlugin;
```

### Lifecycle Hooks

1. **onInit**: Called immediately after component creation
2. **onAdd(evt)**: Called when component is added to viewer (`evt.target` is the viewer)
3. **onRender**: Called when component should render to DOM

### Accessing the Viewer

The viewer instance is provided through the `onAdd` event:

```javascript
onAdd(evt) {
  viewer = evt.target;
  
  // Now you can access viewer methods:
  const map = viewer.getMap();
  const layers = viewer.getLayers();
  const projection = viewer.getProjection();
}
```

## Creating Your First Plugin

Let's create a simple "Map Info" plugin that displays the current map center and zoom level.

### Step 1: Project Structure

```
my-map-info-plugin/
├── src/
│   └── mapinfo.js
├── package.json
├── vite.config.js
└── README.md
```

### Step 2: Plugin Code

Create `src/mapinfo.js`:

```javascript
import Origo from 'Origo';

const MapInfo = function MapInfo(options = {}) {
  const {
    buttonText = 'Map Info',
    title = 'Map Information',
    icon = '#ic_info_outline_24px'
  } = options;

  let viewer;
  let mapMenu;
  let menuItem;
  let modal;

  const getMapInfo = () => {
    const map = viewer.getMap();
    const view = map.getView();
    const center = view.getCenter();
    const zoom = view.getZoom();
    const projection = view.getProjection().getCode();

    return `
      <div class="o-map-info">
        <p><strong>Center:</strong> ${center[0].toFixed(2)}, ${center[1].toFixed(2)}</p>
        <p><strong>Zoom:</strong> ${zoom.toFixed(2)}</p>
        <p><strong>Projection:</strong> ${projection}</p>
      </div>
    `;
  };

  const showInfo = function showInfo() {
    modal = Origo.ui.Modal({
      title,
      content: getMapInfo(),
      target: viewer.getId()
    });
    this.addComponent(modal);
  };

  return Origo.ui.Component({
    name: 'mapinfo',
    
    onAdd(evt) {
      viewer = evt.target;
      
      // Add to map menu
      mapMenu = viewer.getControlByName('mapmenu');
      
      if (mapMenu) {
        menuItem = mapMenu.MenuItem({
          click: showInfo,
          icon,
          title: buttonText
        });
        this.addComponent(menuItem);
      }
      
      this.render();
    },
    
    onRender() {
      if (mapMenu && menuItem) {
        mapMenu.appendMenuItem(menuItem);
      }
      this.dispatch('render');
    }
  });
};

export default MapInfo;
```

### Step 3: Build Configuration (Vite)

Create `vite.config.js`:

```javascript
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    lib: {
      entry: 'src/mapinfo.js',
      name: 'MapInfo',
      fileName: 'mapinfo',
      formats: ['es', 'umd']
    },
    rollupOptions: {
      // Externalize Origo - it will be provided by the host application
      external: ['Origo'],
      output: {
        globals: {
          Origo: 'Origo'
        }
      }
    }
  }
});
```

### Step 4: Package.json

```json
{
  "name": "origo-mapinfo-plugin",
  "version": "1.0.0",
  "description": "A simple plugin to display map information",
  "main": "dist/mapinfo.umd.js",
  "module": "dist/mapinfo.es.js",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "peerDependencies": {
    "ol": "^10.6.0"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

### Step 5: Build the Plugin

```bash
npm run build
```

This creates:
- `dist/mapinfo.es.js` - ES module format
- `dist/mapinfo.umd.js` - UMD format (for browsers)

### Step 6: Use the Plugin

There are two ways to use your plugin:

#### Method 1: Configure in Config File (Recommended)

In your Origo application's HTML:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <link href="css/style.css" rel="stylesheet">
  <title>Origo with MapInfo Plugin</title>
</head>
<body>
  <div id="app-wrapper"></div>
  <script src="js/origo.js"></script>
  <script src="plugins/mapinfo.umd.js"></script>
  <script>
    var origo = Origo('index.json', {
      controls: {
        mapinfo: MapInfo
      }
    });
  </script>
</body>
</html>
```

In your Origo config file (`index.json`):

```json
{
  "controls": [
    {
      "name": "mapinfo",
      "options": {
        "buttonText": "Show Map Info",
        "title": "Current Map Information"
      }
    }
  ]
}
```

#### Method 2: Programmatic Initialization

You can also initialize the plugin programmatically after Origo loads:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <link href="css/style.css" rel="stylesheet">
  <title>Origo with MapInfo Plugin</title>
</head>
<body>
  <div id="app-wrapper"></div>
  <script src="js/origo.js"></script>
  <script src="plugins/mapinfo.umd.js"></script>
  <script>
    var origo = Origo('index.json');
    origo.on('load', function(viewer) {
      var mapinfo = MapInfo({
        buttonText: 'Show Map Info',
        title: 'Current Map Information'
      });
      viewer.addComponent(mapinfo);
    });
  </script>
</body>
</html>
```

**Note**: If your plugin includes CSS, make sure to include it as well:
```html
<link href="plugins/mapinfo.css" rel="stylesheet">
```

## Common Patterns

### Adding a Button to the Map Menu

```javascript
onAdd(evt) {
  viewer = evt.target;
  const mapMenu = viewer.getControlByName('mapmenu');
  
  const menuItem = mapMenu.MenuItem({
    click() {
      // Handle click
      console.log('Menu item clicked');
    },
    icon: '#ic_layers_24px',
    title: 'My Tool'
  });
  
  this.addComponent(menuItem);
  this.render();
}

onRender() {
  mapMenu.appendMenuItem(menuItem);
  this.dispatch('render');
}
```

### Adding a Button to the Map Tools (Toolbar)

```javascript
import Origo from 'Origo';

onAdd(evt) {
  viewer = evt.target;
  const mapTools = viewer.getMain().getMapTools();
  
  const button = Origo.ui.Button({
    cls: 'o-my-tool padding-small icon-smaller round light box-shadow',
    click() {
      console.log('Toolbar button clicked');
    },
    icon: '#ic_build_24px',
    tooltipText: 'My Tool',
    tooltipPlacement: 'east'
  });
  
  this.addComponent(button);
  this.render();
}

onRender() {
  const mapToolsId = viewer.getMain().getMapTools().getId();
  const el = Origo.ui.dom.html(button.render());
  document.getElementById(mapToolsId).appendChild(el);
  this.dispatch('render');
}
```

### Creating a Modal Dialog

```javascript
import Origo from 'Origo';

const showModal = function showModal() {
  const modal = Origo.ui.Modal({
    title: 'My Dialog',
    content: '<p>Dialog content here</p>',
    target: viewer.getId(),
    style: 'max-width: 500px;'
  });
  
  this.addComponent(modal);
};
```

### Working with Layers

```javascript
// Get all layers
const layers = viewer.getLayers();

// Get a specific layer
const myLayer = viewer.getLayer('myLayerName');

// Add a new layer
const newLayer = viewer.addLayer({
  name: 'myNewLayer',
  source: 'mySource',
  style: 'myStyle',
  type: 'WMS'
});

// Remove a layer
viewer.removeLayer('layerName');

// Listen for layer changes
viewer.on('add:layer', (evt) => {
  console.log('Layer added:', evt.layer);
});
```

### Working with Features

```javascript
// Get the selection manager
const selectionManager = viewer.getSelectionManager();

// Get selected features
const selectedFeatures = selectionManager.getSelectedItems();

// Add features to selection
selectionManager.addOrHighlightSelectedItems(features);

// Clear selection
selectionManager.clearSelection();

// Listen for selection changes
selectionManager.on('changeselection', (evt) => {
  console.log('Selection changed:', evt);
});
```

### Interacting with the Map

```javascript
const map = viewer.getMap();
const view = map.getView();

// Get/set center
const center = view.getCenter();
view.setCenter([x, y]);

// Get/set zoom
const zoom = view.getZoom();
view.setZoom(12);

// Fit to extent
view.fit([minX, minY, maxX, maxY], {
  padding: [50, 50, 50, 50],
  duration: 1000
});

// Add map interactions
import { Draw } from 'ol/interaction';

const drawInteraction = new Draw({
  type: 'Point',
  source: myVectorSource
});

map.addInteraction(drawInteraction);
```

### Using OpenLayers

Origo exposes OpenLayers through the `Origo.ol` namespace:

```javascript
// Access in your plugin
const { Point } = Origo.ol.geom;
const { Vector } = Origo.ol.layer;
const { VectorSource } = Origo.ol.source;
const { Style, Circle, Fill, Stroke } = Origo.ol.style;

// Create a simple marker
const markerFeature = new Origo.ol.Feature({
  geometry: new Point([x, y])
});

const markerStyle = new Style({
  image: new Circle({
    radius: 7,
    fill: new Fill({ color: 'red' }),
    stroke: new Stroke({ color: 'white', width: 2 })
  })
});

markerFeature.setStyle(markerStyle);
```

**Note**: If you need OpenLayers features not exposed via `Origo.ol`, you can import OpenLayers directly:

```javascript
import Origo from 'Origo';
import { getCenter } from 'ol/extent';
import TileLayer from 'ol/layer/Tile';

const MyPlugin = function MyPlugin(options = {}) {
  // Plugin code using both Origo and direct OpenLayers imports
};

export default MyPlugin;
```

When importing OpenLayers directly, you don't need to externalize specific ol modules in your Vite config since Origo already includes OpenLayers. However, be aware that this may slightly increase your bundle size.

## Extension Points

### Viewer Events

The viewer fires several events you can listen to:

```javascript
viewer.on('load', () => {
  console.log('Viewer loaded');
});

viewer.on('add:layer', (evt) => {
  console.log('Layer added:', evt.layer);
});

viewer.on('remove:layer', (evt) => {
  console.log('Layer removed:', evt.layer);
});

viewer.on('toggleClickInteraction', (evt) => {
  console.log('Click interaction toggled:', evt);
});
```

### Map Events

```javascript
const map = viewer.getMap();

map.on('click', (evt) => {
  console.log('Map clicked at:', evt.coordinate);
});

map.on('moveend', () => {
  console.log('Map moved');
});

map.on('pointermove', (evt) => {
  // Handle pointer movement
});
```

### Custom Events

You can dispatch and listen to custom events:

```javascript
// Dispatch a custom event
viewer.dispatch('myplugin:action', { data: 'some data' });

// Listen to custom events
viewer.on('myplugin:action', (evt) => {
  console.log('Custom event received:', evt.data);
});
```

## API Reference

### Viewer API

The viewer instance provides the following key methods:

#### Map Access
- `getMap()` - Returns the OpenLayers map instance
- `getMapUtils()` - Returns map utility functions
- `getProjection()` - Returns the current projection
- `getProjectionCode()` - Returns projection code as string

#### Layer Management
- `getLayers()` - Returns all layers
- `getLayer(name)` - Returns a specific layer by name
- `addLayer(config)` - Adds a new layer
- `removeLayer(name)` - Removes a layer
- `getLayersByProperty(property, value)` - Find layers by property

#### Group Management
- `getGroups()` - Returns all layer groups
- `getGroup(name)` - Returns a specific group
- `addGroup(config)` - Adds a new group
- `removeGroup(name)` - Removes a group

#### Control Access
- `getControlByName(name)` - Returns a control by name
- `getControls()` - Returns all controls

#### View Management
- `getCenter()` - Get map center coordinates
- `getZoom()` - Get current zoom level
- `getResolutions()` - Get available resolutions
- `getSize()` - Get map size in pixels

#### Selection
- `getSelectionManager()` - Returns the selection manager
- `getFeatureinfo()` - Returns the feature info component

#### Components
- `getMain()` - Returns the main component
- `getFooter()` - Returns the footer component

#### Utilities
- `getId()` - Returns the viewer's unique ID
- `getUrl()` - Returns the URL of the current map
- `getBaseUrl()` - Returns the base URL

### UI Components

Origo provides several UI components accessible via `Origo.ui`:

#### Component
Base component class:
```javascript
import Origo from 'Origo';

const myComponent = Origo.ui.Component({
  name: 'mycomponent',
  onInit() {},
  onAdd(evt) {},
  onRender() {}
});
```

#### Button
```javascript
import Origo from 'Origo';

const button = Origo.ui.Button({
  cls: 'round light',
  click() { /* handle click */ },
  icon: '#ic_home_24px',
  text: 'Click me',
  tooltipText: 'Tooltip',
  tooltipPlacement: 'east' // 'north', 'south', 'east', 'west'
});
```

#### Modal
```javascript
import Origo from 'Origo';

const modal = Origo.ui.Modal({
  title: 'Modal Title',
  content: '<p>Content</p>',
  target: viewer.getId(),
  style: 'max-width: 600px;'
});
```

#### Element
```javascript
import Origo from 'Origo';

const element = Origo.ui.Element({
  tagName: 'div',
  cls: 'my-class',
  innerHTML: 'Content',
  style: 'background: red;'
});
```

#### dom Utilities
```javascript
import Origo from 'Origo';

// Convert HTML string to element
const el = Origo.ui.dom.html('<div>Hello</div>');

// Create element
const div = Origo.ui.dom.createElement('div', { cls: 'my-class' });
```

### OpenLayers API

Origo exposes OpenLayers modules via `Origo.ol`:

- `Origo.ol.geom` - Geometry classes (Point, LineString, Polygon, etc.)
- `Origo.ol.layer` - Layer classes (Vector, Tile, Image, etc.)
- `Origo.ol.source` - Source classes (Vector, XYZ, TileWMS, etc.)
- `Origo.ol.style` - Style classes (Style, Fill, Stroke, Circle, Icon, etc.)
- `Origo.ol.Feature` - Feature class
- `Origo.ol.Collection` - Collection class
- `Origo.ol.Overlay` - Overlay class
- `Origo.ol.format` - Format classes (GeoJSON, WKT, etc.)
- `Origo.ol.proj` - Projection utilities
- `Origo.ol.interaction` - Interaction classes (Draw, Modify, Select, etc.)

### Utility Functions

#### Origo.Utils
```javascript
import Origo from 'Origo';

// Generate unique ID
const id = Origo.Utils.generateUUID();

// Deep merge objects
const merged = Origo.Utils.deepMerge(obj1, obj2);

// Format numbers
const formatted = Origo.Utils.formatNumber(1234.567, 2);
```

#### Origo.mapUtils
```javascript
import Origo from 'Origo';

// Get features at pixel
const features = Origo.mapUtils.getFeaturesByCoordinate({
  coordinate: [x, y],
  clusterFeatureName: 'clusterLayer',
  map: viewer.getMap(),
  viewer
});

// Get resolution from scale
const resolution = Origo.mapUtils.resolutionFromScale(scale, projection);
```

### Loader API

For showing loading indicators:

```javascript
// Show loader
Origo.Loader.show();

// Hide loader
Origo.Loader.hide();

// Wrap async operation with loader
await Origo.Loader.withLoading(async () => {
  // Your async operation
  await someAsyncTask();
});

// Get inline spinner
const spinner = Origo.Loader.getInlineSpinner();
```

## Localization Support

Origo includes a localization system for multi-language support. Plugins can utilize this to make their UI translatable.

### Accessing Localization

The localization control is passed to your plugin via options:

```javascript
const MyPlugin = function MyPlugin(options = {}) {
  const localization = options.localization;
  
  // Helper function to localize strings
  function localize(key) {
    return localization.getStringByKeys({
      targetParentKey: 'myplugin',
      targetKey: key
    });
  }
  
  return Component({
    name: 'myplugin',
    onAdd(evt) {
      const buttonText = localize('buttonText');
      const title = localize('title');
      // Use localized strings in UI
    }
  });
};
```

### Creating Language Files

Create language files for your plugin:

**locales/en-US.json**:
```json
{
  "myplugin": {
    "buttonText": "My Tool",
    "title": "My Plugin",
    "description": "This is my plugin",
    "messages": {
      "success": "Operation completed successfully",
      "error": "An error occurred"
    }
  }
}
```

**locales/sv-SE.json**:
```json
{
  "myplugin": {
    "buttonText": "Mitt Verktyg",
    "title": "Min Plugin",
    "description": "Detta är min plugin",
    "messages": {
      "success": "Operationen slutfördes",
      "error": "Ett fel uppstod"
    }
  }
}
```

### Registering Language Files

Users need to include your language files in their Origo configuration:

```json
{
  "controls": [
    {
      "name": "localization",
      "options": {
        "localeId": "sv-SE",
        "locales": [
          {
            "localeId": "en-US",
            "url": "locales/en-US.json"
          },
          {
            "localeId": "sv-SE",
            "url": "locales/sv-SE.json"
          },
          {
            "localeId": "en-US",
            "url": "plugins/myplugin-locales/en-US.json"
          },
          {
            "localeId": "sv-SE",
            "url": "plugins/myplugin-locales/sv-SE.json"
          }
        ]
      }
    }
  ]
}
```

### Nested Keys

You can use nested keys for better organization:

```javascript
// Get nested key
const message = localization.getStringByKeys({
  targetParentKey: 'myplugin',
  targetKey: 'messages.success'
});
```

## Building with Vite

Vite is a modern, fast build tool that's recommended for plugin development. It's faster than Webpack and provides a better development experience.

### Basic Vite Setup

1. **Install Vite**:
```bash
npm install --save-dev vite
```

2. **Create vite.config.js**:
```javascript
import { defineConfig } from 'vite';
import path from 'path';

export default defineConfig({
  build: {
    lib: {
      entry: path.resolve(__dirname, 'src/index.js'),
      name: 'MyPlugin',
      fileName: (format) => `my-plugin.${format}.js`,
      formats: ['es', 'umd']
    },
    rollupOptions: {
      // Externalize Origo - it will be provided by the host application
      external: ['Origo'],
      output: {
        globals: {
          Origo: 'Origo'
        }
      }
    }
  },
  server: {
    port: 3000,
    open: '/demo.html'
  }
});
```

### Development Server

Create a demo HTML file for development:

**demo.html**:
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <link href="https://cdn.jsdelivr.net/gh/origo-map/origo@latest/dist/style.css" rel="stylesheet">
  <title>Plugin Development</title>
</head>
<body>
  <div id="app-wrapper"></div>
  <script src="https://cdn.jsdelivr.net/gh/origo-map/origo@latest/dist/origo.min.js"></script>
  <script type="module">
    import MyPlugin from './src/index.js';
    
    const origo = Origo('config.json', {
      controls: {
        myplugin: MyPlugin
      }
    });
  </script>
</body>
</html>
```

Start the development server:
```bash
npm run dev
```

### Production Build

Build for production:
```bash
npm run build
```

This creates:
- `dist/my-plugin.es.js` - For modern bundlers
- `dist/my-plugin.umd.js` - For direct browser use

### Advanced Vite Configuration

For more complex plugins:

```javascript
import { defineConfig } from 'vite';
import path from 'path';

export default defineConfig({
  build: {
    lib: {
      entry: path.resolve(__dirname, 'src/index.js'),
      name: 'MyPlugin',
      formats: ['es', 'umd']
    },
    rollupOptions: {
      // Externalize Origo - provided by the host application
      external: ['Origo'],
      output: {
        // Provide globals for UMD build
        globals: {
          Origo: 'Origo'
        },
        assetFileNames: (assetInfo) => {
          if (assetInfo.name === 'style.css') {
            return 'my-plugin.css';
          }
          return assetInfo.name;
        }
      }
    },
    // Generate source maps for debugging
    sourcemap: true,
    // Minify output
    minify: 'terser'
  },
  // CSS handling
  css: {
    preprocessorOptions: {
      scss: {
        additionalData: `@import "./src/styles/variables.scss";`
      }
    }
  }
});
```

### Including CSS

If your plugin has CSS:

**src/index.js**:
```javascript
import './styles/myplugin.css';
import Origo from 'Origo';

const MyPlugin = function MyPlugin(options = {}) {
  // Plugin code
};

export default MyPlugin;
```

The CSS will be bundled into `dist/style.css` automatically.

## Testing

### Manual Testing

Create a test Origo instance:

1. Clone Origo:
```bash
git clone https://github.com/origo-map/origo.git
cd origo
npm install
```

2. Copy your plugin to the `plugins` folder:
```bash
cp -r ../my-plugin/dist plugins/my-plugin
```

3. Add your plugin to the HTML:
```html
<script src="plugins/my-plugin/my-plugin.umd.js"></script>
```

4. Configure it in `index.json`:
```json
{
  "controls": [
    {
      "name": "myplugin",
      "options": {}
    }
  ]
}
```

5. Start Origo:
```bash
npm start
```

### Automated Testing

For unit tests, use a testing framework like Vitest:

```bash
npm install --save-dev vitest @vitest/ui jsdom
```

**vitest.config.js**:
```javascript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: true
  }
});
```

**Example test** (`src/__tests__/myplugin.test.js`):
```javascript
import { describe, it, expect, vi } from 'vitest';
import MyPlugin from '../myplugin.js';

describe('MyPlugin', () => {
  it('should create a component', () => {
    const plugin = MyPlugin({});
    expect(plugin).toBeDefined();
    expect(plugin.name).toBe('myplugin');
  });

  it('should call onAdd when added', () => {
    const plugin = MyPlugin({});
    const onAddSpy = vi.spyOn(plugin, 'onAdd');
    
    plugin.dispatch('add', { target: {} });
    
    expect(onAddSpy).toHaveBeenCalled();
  });
});
```

Run tests:
```bash
npm test
```

## Best Practices

### 1. Reuse Origo Dependencies

Always use OpenLayers and other libraries provided by Origo instead of bundling your own:

```javascript
// Good - uses Origo's OpenLayers
const { Point } = Origo.ol.geom;

// Bad - imports own OpenLayers
import Point from 'ol/geom/Point';
```

### 2. Follow Naming Conventions

- Plugin names: lowercase with hyphens (`my-plugin`)
- Component names: camelCase (`myPlugin`)
- CSS classes: prefix with `o-` (`o-my-plugin`)
- Events: namespace with plugin name (`myplugin:action`)

### 3. Make Plugins Configurable

Allow users to configure your plugin:

```javascript
const MyPlugin = function MyPlugin(options = {}) {
  const {
    buttonText = 'Default Text',
    icon = '#ic_default_24px',
    position = 'menu',
    enabled = true
  } = options;
  
  // Use configured values
};
```

### 4. Handle Errors Gracefully

```javascript
onAdd(evt) {
  try {
    viewer = evt.target;
    
    const mapMenu = viewer.getControlByName('mapmenu');
    if (!mapMenu) {
      console.warn('MyPlugin: mapmenu not found');
      return;
    }
    
    // Continue setup
  } catch (error) {
    console.error('MyPlugin initialization error:', error);
  }
}
```

### 5. Clean Up Resources

Remove event listeners and interactions when the plugin is removed:

```javascript
const MyPlugin = function MyPlugin(options = {}) {
  let viewer;
  let mapClickListener;
  let drawInteraction;
  
  return Component({
    name: 'myplugin',
    
    onAdd(evt) {
      viewer = evt.target;
      const map = viewer.getMap();
      
      // Add listeners
      mapClickListener = map.on('click', handleClick);
      
      // Add interactions
      drawInteraction = new Draw({ type: 'Point' });
      map.addInteraction(drawInteraction);
    },
    
    onClear() {
      // Clean up
      if (mapClickListener) {
        Origo.ol.Observable.unByKey(mapClickListener);
      }
      if (drawInteraction) {
        viewer.getMap().removeInteraction(drawInteraction);
      }
    }
  });
};
```

### 6. Support Localization

Make your plugin translatable:

```javascript
const localization = options.localization;

function localize(key) {
  if (!localization) {
    return key; // Fallback if localization not available
  }
  return localization.getStringByKeys({
    targetParentKey: 'myplugin',
    targetKey: key
  });
}
```

### 7. Document Your Plugin

Create a comprehensive README.md:

```markdown
# My Plugin

Brief description of what the plugin does.

## Installation

Instructions for installing the plugin.

## Configuration

Example configuration:

\`\`\`json
{
  "name": "myplugin",
  "options": {
    "option1": "value1"
  }
}
\`\`\`

## Options

- `option1` (string): Description
- `option2` (boolean): Description

## Examples

Usage examples

## License

License information
```

### 8. Version Your Plugin

Use semantic versioning (semver):
- Major version: Breaking changes
- Minor version: New features (backward compatible)
- Patch version: Bug fixes

### 9. Keep It Simple

- Don't over-engineer
- Start with the minimum viable feature
- Add complexity only when needed
- Follow the single responsibility principle

### 10. Test in Multiple Browsers

Test your plugin in:
- Chrome
- Firefox
- Safari
- Edge

## Examples

### Example 1: Layer Toggler

A simple plugin to toggle layer visibility:

```javascript
import Origo from 'Origo';

const LayerToggler = function LayerToggler(options = {}) {
  const { layerName, buttonText = 'Toggle Layer' } = options;
  let viewer;
  let layer;
  let button;

  const toggleLayer = () => {
    if (layer) {
      const visible = layer.getVisible();
      layer.setVisible(!visible);
      button.setState(!visible ? 'active' : 'initial');
    }
  };

  return Origo.ui.Component({
    name: 'layertoggler',
    
    onAdd(evt) {
      viewer = evt.target;
      layer = viewer.getLayer(layerName);
      
      if (!layer) {
        console.warn(`Layer ${layerName} not found`);
        return;
      }

      button = Origo.ui.Button({
        cls: 'round light',
        text: buttonText,
        click: toggleLayer,
        state: layer.getVisible() ? 'active' : 'initial'
      });

      this.addComponent(button);
      this.render();
    },
    
    onRender() {
      const mapToolsId = viewer.getMain().getMapTools().getId();
      const el = Origo.ui.dom.html(button.render());
      document.getElementById(mapToolsId).appendChild(el);
      this.dispatch('render');
    }
  });
};

export default LayerToggler;
```

### Example 2: Coordinate Display

Display current mouse coordinates:

```javascript
import Origo from 'Origo';

const CoordinateDisplay = function CoordinateDisplay(options = {}) {
  const { decimals = 2 } = options;
  let viewer;
  let coordinateElement;
  let mapMoveListener;

  const formatCoordinate = (coord) => {
    return `X: ${coord[0].toFixed(decimals)}, Y: ${coord[1].toFixed(decimals)}`;
  };

  return Origo.ui.Component({
    name: 'coordinatedisplay',
    
    onAdd(evt) {
      viewer = evt.target;
      const map = viewer.getMap();

      coordinateElement = Origo.ui.Element({
        tagName: 'div',
        cls: 'o-coordinate-display',
        style: 'position: absolute; bottom: 10px; right: 10px; background: white; padding: 5px; border-radius: 3px;'
      });

      mapMoveListener = map.on('pointermove', (e) => {
        const coord = e.coordinate;
        const el = document.getElementById(coordinateElement.getId());
        if (el) {
          el.innerHTML = formatCoordinate(coord);
        }
      });

      this.addComponent(coordinateElement);
      this.render();
    },
    
    onRender() {
      const el = Origo.ui.dom.html(coordinateElement.render());
      document.getElementById(viewer.getId()).appendChild(el);
      this.dispatch('render');
    },
    
    onClear() {
      if (mapMoveListener) {
        Origo.ol.Observable.unByKey(mapMoveListener);
      }
    }
  });
};

export default CoordinateDisplay;
```

### Example 3: Feature Filter

Filter features in a layer:

```javascript
import Origo from 'Origo';

const FeatureFilter = function FeatureFilter(options = {}) {
  const {
    layerName,
    attributeName,
    title = 'Filter Features'
  } = options;
  
  let viewer;
  let layer;
  let allFeatures;

  const createFilterUI = () => {
    // Get unique values
    const source = layer.getSource();
    allFeatures = source.getFeatures();
    const uniqueValues = new Set();
    
    allFeatures.forEach(feature => {
      const value = feature.get(attributeName);
      if (value) uniqueValues.add(value);
    });

    // Create checkboxes
    let html = '<div class="o-feature-filter">';
    uniqueValues.forEach(value => {
      html += `
        <label>
          <input type="checkbox" value="${value}" checked>
          ${value}
        </label><br>
      `;
    });
    html += '</div>';

    return html;
  };

  const applyFilter = (selectedValues) => {
    const source = layer.getSource();
    source.clear();
    
    const filteredFeatures = allFeatures.filter(feature => {
      const value = feature.get(attributeName);
      return selectedValues.includes(value);
    });
    
    source.addFeatures(filteredFeatures);
  };

  const showFilterDialog = function showFilterDialog() {
    const content = createFilterUI();
    const modal = Origo.ui.Modal({
      title,
      content,
      target: viewer.getId()
    });

    this.addComponent(modal);

    // Add event listeners after render
    setTimeout(() => {
      const checkboxes = document.querySelectorAll('.o-feature-filter input');
      checkboxes.forEach(checkbox => {
        checkbox.addEventListener('change', () => {
          const selectedValues = Array.from(checkboxes)
            .filter(cb => cb.checked)
            .map(cb => cb.value);
          applyFilter(selectedValues);
        });
      });
    }, 100);
  };

  return Origo.ui.Component({
    name: 'featurefilter',
    
    onAdd(evt) {
      viewer = evt.target;
      layer = viewer.getLayer(layerName);

      if (!layer) {
        console.warn(`Layer ${layerName} not found`);
        return;
      }

      const mapMenu = viewer.getControlByName('mapmenu');
      if (mapMenu) {
        const menuItem = mapMenu.MenuItem({
          click: showFilterDialog,
          icon: '#ic_filter_list_24px',
          title
        });
        this.addComponent(menuItem);
      }

      this.render();
    },
    
    onRender() {
      const mapMenu = viewer.getControlByName('mapmenu');
      if (mapMenu) {
        mapMenu.appendMenuItem(this.getComponents()[0]);
      }
      this.dispatch('render');
    }
  });
};

export default FeatureFilter;
```

## Real-World Plugin Examples

Study these existing plugins for more examples:

### Simple Plugins
- **[lmsearch-plugin](https://github.com/origo-map/lmsearch-plugin)** - Search integration
- **[multiselect-plugin](https://github.com/origo-map/multiselect-plugin)** - Feature selection tools
- **[barebone-plugin](https://github.com/origo-map/barebone-plugin)** - Minimal template

### Advanced Plugins
- **[layermanager](https://github.com/origo-map/layermanager)** - Complex UI with layer tree
- **[elevation-profile-plugin-v2](https://github.com/jokd/elevation-profile-plugin-v2)** - Chart integration
- **[ek-filter-plugin](https://github.com/Eskilstuna-kommun/ek-filter-plugin)** - Advanced filtering

## Getting Help

- **Documentation**: [https://origo-map.github.io/origo-documentation/](https://origo-map.github.io/origo-documentation/)
- **Discord**: [origo.map Discord server](https://discord.gg/NWRAkWAXQ3)
- **GitHub Issues**: [origo-map/origo/issues](https://github.com/origo-map/origo/issues)
- **Developer Seminars**: [Developing in Origo part 3 - plugins](https://docs.google.com/presentation/d/13e38b81OnGhhud2t4cDxux5vXzGZYFLO/edit?usp=sharing&ouid=116035513551791915749&rtpof=true&sd=true)

## Contributing

If you've created a plugin that might be useful to others, consider:

1. Publishing it to npm
2. Adding it to the [PLUGINS.md](PLUGINS.md) list
3. Sharing it on the Discord server
4. Creating a demo repository

## License

When creating plugins, ensure your license is compatible with Origo's BSD-2-Clause license. Common compatible licenses include:
- MIT
- BSD-2-Clause
- BSD-3-Clause
- Apache-2.0

---

**Happy plugin development!** 🎉

If you have questions or need help, don't hesitate to reach out to the Origo community.
