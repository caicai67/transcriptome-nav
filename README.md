# Transcriptome Navigation

This is an [Observable Framework](https://observablehq.com/framework) project. To start the local preview server, run:

```
npm run dev
```

Then visit <http://localhost:3000> to preview your project.

For more, see <https://observablehq.com/framework/getting-started>.

## Project structure

A typical Framework project looks like this:

```ini
.
├─ src
│  ├─ components
│  │  └─ timeline.js           # an importable module
│  ├─ data
│  │  ├─ launches.csv.js       # a data loader
│  │  └─ events.json           # a static data file
│  ├─ example-dashboard.md     # a page
│  ├─ example-report.md        # another page
│  └─ index.md                 # the home page
├─ .gitignore
├─ observablehq.config.js      # the project config file
├─ package.json
└─ README.md
```

**`src`** - This is the “source root” — where your source files live. Pages go here. Each page is a Markdown file. Observable Framework uses [file-based routing](https://observablehq.com/framework/routing), which means that the name of the file controls where the page is served. You can create as many pages as you like. Use folders to organize your pages.

**`src/index.md`** - This is the home page for your site. You can have as many additional pages as you’d like, but you should always have a home page, too.

**`src/data`** - You can put [data loaders](https://observablehq.com/framework/loaders) or static data files anywhere in your source root, but we recommend putting them here.

**`src/components`** - You can put shared [JavaScript modules](https://observablehq.com/framework/javascript/imports) anywhere in your source root, but we recommend putting them here. This helps you pull code out of Markdown files and into JavaScript modules, making it easier to reuse code across pages, write tests and run linters, and even share code with vanilla web applications.

**`observablehq.config.js`** - This is the [project configuration](https://observablehq.com/framework/config) file, such as the pages and sections in the sidebar navigation, and the project’s title.

## Command reference

| Command           | Description                                              |
| ----------------- | -------------------------------------------------------- |
| `npm install`            | Install or reinstall dependencies                        |
| `npm run dev`        | Start local preview server                               |
| `npm run build`      | Build your static site, generating `./dist`              |
| `npm run deploy`     | Deploy your project to Observable                        |
| `npm run clean`      | Clear the local data loader cache                        |
| `npm run observable` | Run commands like `observable help`                      |


# References
The source data currently used for this project is from the CosMx SMI
NSCLC FFPE Dataset: 
https://nanostring.com/products/cosmx-spatial-molecular-imager/ffpe-dataset/nsclc-ffpe-dataset/


# Guides

## Deployment

- build using `npm run build`
- Commit changes to the main branch (these are generated from the build)
- Wait for github pages to update

## Run locally 

To make changes to the app, you will want to run it locally.

- open two terminal/command windows and navigate to the project folder
- in one, run `node server.js`
- in the other, run `npm run dev`

A window will appear in the browser with the app

# Under the hood

## Deployment process

Putting changes on the public website is called a deployment. The deployment process is controlled by two files:
`/transcriptome-nav/package.json` and `/transcriptome-nav/.github/workflows/publish.yml`.

`package.json` creates the build packages, which are what github pages uses to display the website. These build files are placed in a folder called `dist`. In this project, `package.json` was modified to take the image files we included for the cell types, and copy them into the `dist` folder so they will be deployed along with the code. 

`Package.json` is applied when we run `npm run build`.

`publish.yml` allows us to have more control over how the files are interpreted in github pages. Github pages prefers to pull files from a `docs` folder. In this file, we clear the `docs` folder and copy the `dist` folder contents into the `docs` folder. 

`publish.yml` is applied whenever a change is pushed to the `main` branch.

## Local run

When we run locally, we spin up two servers, one with `node server.js`, and one with `npm run dev`. `node server.js` serves up the images on localhost, ` http://localhost:3000`. `npm run dev` serves the app on the localhost ip address, `http://127.0.0.1:3000/`.





