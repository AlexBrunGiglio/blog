---
title: "Créer une librairie de composants React"
date: 2022-09-09T20:43:22+02:00
draft: false
---
- Initialisation projet
- Implémentation rollup
- Implémentation CSS / Scss
- Optimisation de la librairie
- Implémentation des tests (JEST)
- Ajout d'une documentation de la lib (Storybook)

## Création projet 
```sh
yarn init
```

```sh
yarn add react typescript @types/react --save-dev
```

Il faudra ensuite créer un dossier src et dans celui-ci un dossier components.
Le premier composant sera un bouton. 

```
.
├── src
│   ├── components
|   │   ├── Button
|   |   │   ├── Button.tsx
|   |   │   └── index.ts
|   │   └── index.ts
│   └── index.ts
├── package.json
└── yarn.lock
```

`src/components/Button/Button.tsx` :
```jsx
import React from "react";

export interface ButtonProps {
  label: string;
}

const Button = (props: ButtonProps) => {
  return <button>{props.label}</button>;
};

export default Button;
```

`src/components/Button/index.ts` :
```ts
export { default } from "./Button";
```

`src/components/index.ts` :
```ts
export { default as Button } from "./Button";
```

`src/index.ts` :
```ts
export * from './components';
```

### Ajout de Typescript

```sh
npx tsc --init
# Cela va générer un fichier tsconfig.json
```

Modification du `tsconfig.json` :
```json
{
  "compilerOptions": {
    "target": "es5", 
    "esModuleInterop": true, 
    "forceConsistentCasingInFileNames": true,
    "strict": true, 
    "skipLibCheck": true,
    "jsx": "react", 
    "module": "ESNext",  
    "declaration": true,
    "declarationDir": "types",
    "sourceMap": true,
    "outDir": "dist",
    "moduleResolution": "node",
    "allowSyntheticDefaultImports": true,
    "emitDeclarationOnly": true,
  }
}
```

## Ajout de rollup
```sh
yarn add rollup @rollup/plugin-node-resolve @rollup/plugin-typescript @rollup/plugin-commonjs rollup-plugin-dts --save-dev
# Cela va générer un fichier rollup.config.js
```
`rollup.config.js` :
```js
import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";
import typescript from "@rollup/plugin-typescript";
import dts from "rollup-plugin-dts";

const packageJson = require("./package.json");

export default [
  {
    input: "src/index.ts",
    output: [
      {
        file: packageJson.main,
        format: "cjs",
        sourcemap: true,
      },
      {
        file: packageJson.module,
        format: "esm",
        sourcemap: true,
      },
    ],
    plugins: [
      resolve(),
      commonjs(),
      typescript({ tsconfig: "./tsconfig.json" }),
    ],
  },
  {
    input: "dist/esm/index.d.ts",
    output: [{ file: "dist/index.d.ts", format: "esm" }],
    plugins: [dts()],
  },
];
```

Modification du `package.json` :
```json
...
  "scripts": {
    "rollup": "rollup -c"
  },
   "main": "dist/cjs/index.js",
  "module": "dist/esm/index.js",
  "files": [
    "dist"
  ],
  "types": "dist/index.d.ts"
```

L'architecture du projet devrait ressembler à ça : 
```
.
├── src
│   ├── components
|   │   ├── Button
|   |   │   ├── Button.tsx
|   |   │   └── index.ts
|   │   └── index.ts
│   └── index.ts
├── package.json
├── yarn.lock
├── tsconfig.json
└── rollup.config.js
```

## Ajout de CSS 

Création du fichier `Button.css` dans `src/components/Button/Button.css`

```css
button {
  /* style... */
}
```

Import du css dans Button.tsx
`src/components/Button/Button.tsx` :
```tsx
import React from "react";
import "./Button.css";

export interface ButtonProps {
  label: string;
}

const Button = (props: ButtonProps) => {
  return <button>{props.label}</button>;
};

export default Button;
```

L'utilisation de Scss sera expliquée après la configuration de storybook.

### Configuration de rollup pour le css

```sh
yarn add rollup-plugin-postcss --save-dev
```


`rollup.config.js` :
```js
import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";
import typescript from "@rollup/plugin-typescript";
import dts from "rollup-plugin-dts";

// AJOUT
import postcss from "rollup-plugin-postcss";

const packageJson = require("./package.json");

export default [
  {
    input: "src/index.ts",
    output: [
      {
        file: packageJson.main,
        format: "cjs",
        sourcemap: true,
      },
      {
        file: packageJson.module,
        format: "esm",
        sourcemap: true,
      },
    ],
    plugins: [
      resolve(),
      commonjs(),
      typescript({ tsconfig: "./tsconfig.json" }),

      // AJOUT
      postcss(), 
    ],
  },
  {
    input: "dist/esm/types/index.d.ts",
    output: [{ file: "dist/index.d.ts", format: "esm" }],
    plugins: [dts()],

    // AJOUT
    external: [/\.css$/],
  },
];
```

## Optimisation de la librairie 
Etape 1 : ajout du plugin Terser : permet de minifier le bundle généré et de réduire la taille des fichiers.


Etape 2 : mettre à jour certaines de nos dépendances en peerDependencies. (Exemple : si dans un projet React on install la librairie, il faut éviter que la librairie install en doublon React)


```sh
yarn add rollup-plugin-peer-deps-external rollup-plugin-terser --save-dev
```

`rollup.config.js` : 
```js
// imports
...
import { terser } from "rollup-plugin-terser";
import peerDepsExternal from 'rollup-plugin-peer-deps-external';

|
└── 
    plugins: [
        // NEW
        peerDepsExternal(),

        resolve(),
        commonjs(),
        typescript({ tsconfig: "./tsconfig.json" }),
        postcss(),

        // NEW
        terser(),
        ],
```

Dans `package.json` définir les librairies considéré comme des peerDependencies : 
```json
{
  "devDependencies": {
    "@rollup/plugin-commonjs": "^21.0.1",
    "@rollup/plugin-node-resolve": "^13.0.6",
    "@rollup/plugin-typescript": "^8.3.0",
    "@types/react": "^17.0.34",
    "rollup": "^2.60.0",
    "rollup-plugin-dts": "^4.0.1",
    "rollup-plugin-peer-deps-external": "^2.2.4",
    "rollup-plugin-postcss": "^4.0.1",
    "rollup-plugin-terser": "^7.0.2",
    "typescript": "^4.4.4"
  },
  "peerDependencies": {
    "react": "^17.0.2"
    [...]
  },
}
```

## Implémentation des tests

```sh
yarn add @testing-library/react jest @types/jest --save-dev
```


Création de `src/components/Button/Button.test.tsx` :

```tsx
import React from "react";
import { render } from "@testing-library/react";

import Button from "./Button";

describe("Button", () => {
  test("renders the Button component", () => {
    render(<Button label="Hello world!" />);
  });
});
```

A la racine du projet création de `jest.config.js` :
```js
module.exports = {
  testEnvironment: "jsdom",
};
```

Ajout du script de test dans `package.json` :

```json
{
  "scripts": {
    "rollup": "rollup -c",
    "test": "jest"
  },
  ...
}
```

Ajout de Babel dans le projet afin que Jest puisse comprendre tsx. 
```sh
yarn add @babel/core @babel/preset-env @babel/preset-react @babel/preset-typescript babel-jest --save-dev
```

Pour que Jest puisse comprendre le CSS il faut ajouter 
```sh
yarn add identity-obj-proxy --save-dev 
```


Création de `babel.config.js` :
```js
module.exports = {
  presets: [
    "@babel/preset-env",
    "@babel/preset-react",
    "@babel/preset-typescript",
  ],
};
```

Modifications de `jest.config.js` :
```js
module.exports = {
  testEnvironment: "jsdom",
  moduleNameMapper: {
    ".(css|less|scss)$": "identity-obj-proxy",
  },
};
```

## Ajout de Storybook
Installation storybook :
```sh
npx sb init --builder webpack5
```

Cela va créer un dossier `.storybook` et un fichier `main.js` à la racine du projet.

Création d'un fichier `Button.stories.tsx` dans `src/components/Button` :
```tsx
import React from "react";
import { ComponentStory, ComponentMeta } from "@storybook/react";
import Button from "./Button";


export default {
  title: "ReactComponentLibrary/Button",
  component: Button,
} as ComponentMeta<typeof Button>;


const Template: ComponentStory<typeof Button> = (args) => <Button {...args} />;

export const HelloWorld = Template.bind({});
HelloWorld.args = {
  label: "Hello world!",
};

export const ClickMe = Template.bind({});
ClickMe.args = {
  label: "Click me!",
};
```

Dans le `tsconfig.json`, ajouter dans compilerOptions : 
```json
{
  "compilerOptions": {
   ...
  },
    "exclude": [
    "src/stories",
    "**/*.stories.tsx"
  ]
}
```

En cas d'erreur il faudra peut être installer tslib :
```sh
yarn add -D tslib
```

