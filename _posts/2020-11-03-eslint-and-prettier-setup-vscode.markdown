---
layout: post
title: "ESLint and Prettier setup for VSCode"
date: 2020-11-03 00:00:00 +0530
categories: vscode devtools
---

I have been using ESLint for linting and fixing my javascript for a long time, but lately, it has been giving me a lot of trouble, so I started looking for an alternative and came across prettier. I used prettier earlier as well but I was not ready to give up my ESLint workflow as it worked fine back then.

## ESLint and Prettier Primer

Before diving into the configuration, let’s understand what these tools are used for.

### ESLint

ESLint is a code analysis tool that finds and reports problems in our code. We set up a bunch of rules in our `.eslintrc.*` file and ESlint makes sure our code follows those rules.

**Example config file**

```json
// .eslintrc.json
{
  "env": {
    "commonjs": true,
    "es2021": true,
    "node": true
  },
  "extends": ["eslint:recommended"],
  "parserOptions": {
    "ecmaVersion": 12
  },
  "rules": {}
}
```

This is a very basic config file but you can find more info about various rules and config options [here](https://eslint.org/docs/rules/).

### Prettier

Prettier is a code formatter, it formats your code according to the rules you specify in the prettier config file.

**Example config file**

```json
// .prettierrc
{
  "trailingComma": "es5",
  "tabWidth": 4,
  "semi": true,
  "singleQuote": false
}
```

Again this is a very basic config file you can find more config options by following [this](https://prettier.io/docs/en/options.html) link.

## Configuration

To get started first we need to install [Prettier](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode) and [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint) extensions from the VSCode [marketplace](https://marketplace.visualstudio.com/).

### Configuring ESLint

From your project root run the following command.

```shell
$ npx eslint --init
```

The configuration wizard will ask a few questions to setup your config file.

### Prettier Configuration

Install Prettier in your project locally(recommended).

```shell
$ yarn add -D prettier --exact
```

Or

```shell
$ npm i -D prettier --save-exact
```

the `--exact` flag pins prettier to a particular version.

### Integrating Prettier with ESLint

So far we have setup Prettier and ESLint they both work fine on their own but sometimes they interfere with each other, let's fix that.

Following Prettier [docs](https://prettier.io/docs/en/integrating-with-linters.html#disable-formatting-rules), we need to install `eslint-config-prettier`.

Install `eslint-config-prettier`.

```shell
$ yarn add -D eslint-config-prettier
```

Then, add `eslint-config-prettier` to the "extends" array in your `.eslintrc.*` file. Make sure to put it last, so it gets the chance to override other configs.

```json
// .eslintrc.json
{
  "env": {
    "commonjs": true,
    "es2021": true,
    "node": true
  },
  "extends": ["eslint:recommended", "prettier"],
  "parserOptions": {
    "ecmaVersion": 12
  },
  "rules": {}
}
```

### Updating VSCode Settings

To finalize our config we need to tell VSCode to use Prettier as a formatter. Add the following to your VSCode settings.

```json
{
  // ...
  "editor.defaultFormatter": "esbenp.prettier-vscode"
}
```

#### Format On Save

Enable format on save by adding the following to your config.

```json
{
  // ...
  "editor.formatOnSave": true
}
```

## Conclusion

Setting up your dev environment is very useful, and tools like Prettier and ESLint can help your code stay consistent across projects and while working with teams.

If you encounter some problem, reach out to me via [twitter](https://twitter.com/thetinygoat), I would love to help you :)
