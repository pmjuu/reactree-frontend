🇰🇷 [Korean README](./README-Korean.md)

# Reactree

<div align=center>

### _" React + Tree "_
This is an app service that visualizes the **component hierarchy** of a **React** project in a **tree** structure.

</div>

<br>

# Table of Contents

- [Preview](#Preview)
- [Motivation](#Motivation)
- [Challenges](#Challenges)
  - [1. How to parse user code?](#1-how-to-parse-user-code)
    - [Extracting data from React Fiber](#extracting-data-from-react-fiber)
  - [2. How to run internal Electron app functions from the user directory?](#2-how-to-run-internal-electron-app-functions-from-the-user-directory)
    - [Accessing reactree functions from the user directory via SymLink](#accessing-reactree-functions-from-the-user-directory-via-symlink)
  - [3. How to transmit rootFiberNode from the user directory?](#3-how-to-transmit-rootfibernode-from-the-user-directory)
  - [4. How to improve data transmission stability in large user projects?](#4-how-to-improve-data-transmission-stability-in-large-user-projects)
- [Tech stacks](#Tech-stacks)
- [Features](#Features)
- [Timeline](#Timeline)

<br>

# Preview

🔽 Rendering the tree structure after folder selection
![reactreeGIF1](https://user-images.githubusercontent.com/50537876/228586806-b776bc89-8750-49f8-8a9d-8f969f12b7a7.gif)
<br><br>
🔽 Zoom in/out of the tree structure, adjust the slider bar, mouse events on nodes (hovering, clicking)
![reactreeGIF2](https://user-images.githubusercontent.com/50537876/228586858-d02b2f78-151e-4ee9-b837-433cb2b20b17.gif)

<br>

# Motivation

When I first started learning React or when I read someone else's React project code for the first time, it took time to understand the overall component structure. <br>
I thought, "Wouldn't it be helpful for developing with React if we could visualize and display the structure of the rendered components?" This thought led to the start of this project.

<br>

# Challenges

## 1. How to parse user code?

### Attempted Methods

#### Sending an API request to Github to receive the code as a string
  ```js
  // API to retrieve GitHub repository information
  async function getGit(owner, repo, path) {
    const dataResponse = await fetch(`https://api.github.com/repos/${owner}/${repo}/contents/${path}`);
    const data = await dataResponse.json();
    const blobsResponse = await fetch(`https://api.github.com/repos/${owner}/${repo}/git/blobs/${data.sha}`);
    const blobs = await blobsResponse.json();
    console.log(atob(blobs.content));

    return blobs;
  }

  await getGit("pmjuu", "my-workout-manager", "src/App.js");
  ```
  🔽 Console output: It was possible to retrieve the file code from an actual repository as a string.<br>
  <img src="https://github.com/pmjuu/climick-client/assets/50537876/337ff9d6-2811-4436-a010-94570bc7d621" width=400><br>

=> We concluded that it would be too difficult to implement parsing the string into JavaScript syntax without using an external library due to the many edge cases, making it hard to achieve within the time limit.

#### Utilizing the ReactDOMServer Object
  ```js
  import React from "react";
  import ReactDOMServer from "react-dom/server";

  // Component to check
  const MyComponent = () => {
    return (
      <div>
        <h1>Hello, world!</h1>
        <p>This is my component.</p>
      </div>
    );
  };

  // Render the component to HTML
  const html = ReactDOMServer.renderToString(<MyComponent />);

  // Extract the DOM Tree from the HTML and visualize it
  const parser = new DOMParser();
  const dom = parser.parseFromString(html, "text/html");
  const tree = dom.body.firstChild;

  console.log(tree);
  ```
  <img src="https://github.com/pmjuu/climick-client/assets/50537876/ab1b7c1d-bfb3-4c2e-ad7c-f245afd90041" width=300><br>

* We could render a component into static markup using the ReactDOMServer object.
* However, components are also rendered as regular tags, so it was necessary to perform additional work to extract the component names separately.

#### Utilizing the `React.Children` property
  ```js
  function buildTree(components) {
    const tree = {
      name: "App",
      children: [],
    }

    React.Children.forEach(components, (component) => {
      if (React.isValidElement(component)) {
        const child = {
          name: component.type.name,
          children: buildTree(component.props.children),
        };
        tree.children.push(child);
      }
    });

    return tree.children.length ? tree.children : null;
  }
  ```
* I was able to extract the names of the components and visualize the hierarchy.
* However, it was limited to the components declared within the `return` statement of the App component.  
  (Regular tags were not displayed.)
* Additionally, extra logic was needed for handling conditional rendering.

#### Using the react-d3-tree package
* I was able to parse the DOM tree and generate a tree structure using the react-d3-tree package.
* However, the main logic of the project became heavily dependent on the package, which led to a lack of technical challenges and made customization difficult.
  <details>
    <summary>Code</summary>

    ```js
    import React, { useState, useEffect } from "react";
    import Tree from "react-d3-tree";

    function DOMTree() {
      const [treeData, setTreeData] = useState({});

      useEffect(() => {
        // Assign the root element of the DOM tree to a variable
        const rootElement = document.documentElement;

        // Parse the DOM tree and create the tree data structure
        function createTree(node) {
          const tree = {};
          tree.name = node.tagName.toLowerCase();
          tree.children = [];

          for (let i = 0; i < node.children.length; i++) {
            const child = node.children[i];
            const childTree = createTree(child);
            tree.children.push(childTree);
          }

          return tree;
        }

        const treeData = createTree(rootElement);
        setTreeData(treeData);
      }, []);

      return (
        <div id="treeWrapper" style={{ width: "159%", height: "100vh" }}>
          <Tree data={treeData} />
        </div>
      );
    }
    ```
  </details>

  <img width=300 src="https://github.com/pmjuu/climick-client/assets/50537876/ca17a12b-d3db-498e-af4d-d7a85ca90e4a">

### Conclusion

* Instead of parsing the code, we use the information of the currently rendered components on the screen.<br>
    -> **Utilizing React Fiber**
* The rendered screen on localhost is displayed in the Electron view by running `npm start` from the user's local directory.<br>
    -> **Utilizing `child_process` in an Electron app**

#### What is React Fiber?

* React Fiber is a new reconciliation algorithm that restructured React's core algorithm in React v16.
* It complements the drawbacks of the old stack reconciler, which executed all tasks synchronously, allowing for concurrency.
* By assigning priorities to certain tasks, it allows parts of the task to be paused and resumed concurrently, enabling incremental rendering.

#### Structure of Fiber

* A fiber is a JavaScript object that contains information about a component and its inputs and outputs.
* It consists of two tree structures: `current` and `workInProgress`.
* Each `fiberNode` has a single linked list structure, pointing to its next node via `return`, `child`, and `sibling` pointers.

### Extracting Data from React Fiber

* In order to visualize the component structure using `d3.js`, the data needed to be structured in a tree format.
* To transform the fiber, which is in the form of a linked list, into a tree structure, I created a recursive function `createNode()` to extract the component tree structure data.
  ```js
  import TreeNode from "./TreeNode";

  const skipTag = [7, 9, 10, 11];

  function createNode(fiberNode, parentTree) {
    if (!fiberNode || Object.keys(fiberNode).length === 0) return null;

    const node = fiberNode.alternate ? fiberNode.alternate : fiberNode;
    const tree = new TreeNode();

    tree.setName(node);
    tree.addProps(node);
    tree.addState(node);

    if (fiberNode.sibling) {
      while (skipTag.includes(fiberNode.sibling.tag)) {
        const originSibling = fiberNode.sibling.sibling;
        fiberNode.sibling = fiberNode.sibling.child;
        fiberNode.sibling.sibling = originSibling;
      }

      const siblingTree = createNode(fiberNode.sibling, parentTree);

      if (siblingTree) {
        parentTree.addChild(siblingTree);
      }
    }

    if (fiberNode.child) {
      while (skipTag.includes(fiberNode.child.tag)) {
        fiberNode.child = fiberNode.child.child;
      }

      const childTree = createNode(fiberNode.child, tree);

      tree.addChild(childTree);
    }

    return tree;
  }
  ```
* Each node is classified by its `tag`, and during the data extraction process, nodes that were unnecessary for visualizing the tree structure were excluded based on their `tag`.
* There are many properties that cause circular references.<br>
  Due to this, it was not possible to execute `JSON.stringify()` during the JSON file generation process, so circular reference properties were excluded.<br>
  🔽 Classification of fiberNode tags
  ```js
  export const FunctionComponent = 0;
  export const ClassComponent = 1;
  export const IndeterminateComponent = 2; // Before we know whether it is function or class
  export const HostRoot = 3; // Root of a host tree. Could be nested inside another node.
  export const HostPortal = 4; // A subtree. Could be an entry point to a different renderer.
  export const HostComponent = 5;
  export const HostText = 6;
  export const Fragment = 7;
  export const Mode = 8;
  export const ContextConsumer = 9;
  export const ContextProvider = 10;
  export const ForwardRef = 11;
  export const Profiler = 12;
  export const SuspenseComponent = 13;
  export const MemoComponent = 14;
  export const SimpleMemoComponent = 15;
  export const LazyComponent = 16;
  export const IncompleteClassComponent = 17;
  export const DehydratedFragment = 18;
  export const SuspenseListComponent = 19;
  export const ScopeComponent = 21;
  export const OffscreenComponent = 22;
  export const LegacyHiddenComponent = 23;
  export const CacheComponent = 24;
  export const TracingMarkerComponent = 25;
  export const HostHoistable = 26;
  export const HostSingleton = 27;
  ```
  <sub>Reference: <a href="https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactWorkTags.js">react Github - type WorkTag</a></sub>
<br>

## 2. How to run internal Electron app functions from the user directory?

### Attempted Methods

* Registering the function we created as an npm package and utilizing it
  - This method is inconvenient because the user would need to install the npm package and manually add code to the project they want to analyze.

### Conclusion

* Utilize `Symlink`.

#### What is a Symlink?

* A symlink (symbolic link) is a type of file in Linux that can reference another file or directory already created, in any file system.

* The syntax to create a symlink is as follows:<br>
  `ln -s <path to the original file/folder> <path to the new link>`

=> To allow the external user directory to reference internal Electron app functions, we decided to use symlink.

#### Referencing the reactree function from the user directory via SymLink

* Create a `symlink` using the Node.js Child process `exec()`.
* In the `src/index.js` file of the user directory, import the `reactree()` function created through the `symlink`.
```js
// public/Electron/ipc-handler.js

exec(
  `ln -s ${userHomePath}/Desktop/reactree-frontend/src/utils/reactree.js ${filePath}/src/reactree-symlink.js`,
  (error, stdout, stderr) => {
    const pathError = getErrorMessage(stderr);

    if (pathError) handleError(view, pathError);
  },
);

const JScodes = `
  // eslint-disable-next-line import/first
  import reactree from "./reactree-symlink";

  setTimeout(() => {
    reactree(root._internalRoot);
  }, 0);
`;

appendFileSync(`${filePath}/src/index.js`, JScodes);
```

<br>

## 3. How to transmit the rootFiberNode from the user directory?

### Attempted Methods

* Assign the fiber data `json` to the `<div id="root">` key value in the user code and retrieve the data by executing `view.webContents.executeJavascript()`
  - Although the functionality worked, I concluded that this approach was impractical due to data size limitations and maintainability issues.

### Conclusion

I discovered that services built with `electron`, like VScode and Slack, download necessary data as `json` files locally.  
Inspired by this, I decided to proceed by downloading the component tree structure data of the user project as a `json` file and reading that file to visualize the component structure.

1. When the user project is run in development mode via `exec()`, the `reactree()` function, referenced via symlink in `index.js`, is executed.
    ```js
    // public/Electron/ipc-handler.js

    execSync(`lsof -i :${portNumber} | grep LISTEN | awk '{print $2}' | xargs kill`);

    exec(
      `PORT=${portNumber} BROWSER=none npm start`,
      { cwd: filePath },
      (error, stdout, stderr) => {
        const startError = getErrorMessage(stderr);

        if (startError) handleError(view, startError);
      },
    );
    ```

2. When the `reactree()` function is executed, the `data.json` file containing the fiber data is downloaded to the user's local storage.
    ```js
    // reactree-frontend/src/utils/reactree.js

    const reactree = rootInternalRoot => {
      try {
        const fiber = deepCopy(rootInternalRoot);
        const fiberJson = JSON.stringify(fiber.current, getCircularReplacer);
        const blob = new Blob([fiberJson], { type: "text/json;charset=utf-8" });
        const url = URL.createObjectURL(blob);

        const link = document.createElement("a");
        link.href = url;
        link.download = "data.json";
        link.click();

        setTimeout(() => {
          URL.revokeObjectURL(url);
        }, 0);

        return undefined;
      } catch (error) {
        return console.error(error);
      }
    };
    ```
3. The Electron `ipc-handler` reads the `data.json` file and sends the data to the renderer process.
    ```js
    // public/Electron/ipc-handler.js

    await waitOn({ resources: [`${userHomePath}/Downloads/data.json`] });

    const fiberFile = readFileSync(
      path.join(`${userHomePath}/Downloads/data.json`),
    );

    mainWindow.webContents.send("node-to-react", JSON.parse(fiberFile));
    ```
<br>

## 4. How to improve data transmission stability in large user projects?

### Attempted Methods

* Initially, I created a link to download the extracted data in JSON format to the local environment with the user's project information. <br>

### Issue Encountered

* While there was no issue with smaller projects, I encountered data truncation issues when dealing with larger projects.
* After investigation, I found that the URI scheme method had data limitations, making it unsuitable for handling large amounts of data.

### Conclusion

* Utilizing the `Blob` object

#### What is a `Blob`?

`Blob` stands for binary large object. As the name suggests, it allows storing data in the form of binary objects.

-> To access `Blob` data, I needed to create a URL that points to the `Blob` object.

1. Using `Blob`'s `createObjectURL()`, I converted the given object into a URL as a DOMString. This URL is automatically revoked when the window is closed.

  ```js
  const blob = new Blob([fiberJson], { type: "text/json;charset=utf-8" });
  const url = URL.createObjectURL(blob);
  ```

2. Create an `<a>` element and set the `href` attribute to the `Blob` URL created above. Then, set the `download` attribute to use it as a link for file downloading.

  ```js
  const link = document.createElement("a");
  link.href = url;
  link.download = "data.json";
  ```

3. Once the download is complete, use `revokeObjectURL()` to invalidate the `Blob` URL and release the resources that are no longer needed. This helps prevent memory leaks.

  ```js
  URL.revokeObjectURL(url);
  ```

<br>

# Tech stacks

### Frontend

- React
- Electron
- Redux, Redux-toolkit
- Styled-Component
- d3
- ESLint

### Test

- Jest, playwright

#### Reasons for Using `Electron`

* Access to system resources
  - You can directly use Node.js APIs within the webview page to access the file system, which is generally not possible in a web browser.
* Utilization of web development technologies
  - Since Chromium is used in the Renderer process (frontend) and Node.js is used in the Main process (backend), existing web development technologies can be utilized.
* Cross-platform compatibility
  - You can develop desktop applications that run on various operating systems such as Windows, macOS, and Linux.

<br>

# Features

- Electron [ Geonhwa 40% / Taewoo 30% / Minju 30% ]
  - Allows access to the user's local storage via system resource access.
  - Runs the program selected by the user through `child_process`.
  - Provides a secure synchronous two-way bridge in isolated contexts (`main`, `renderer`).
- Symlink File Creation [ Geonhwa 50% / Taewoo 30% / Minju 20% ]
  - When the correct folder is selected, it generates a `Symlink` file that allows the `reactree` function to run in the user's project.
- Component Hierarchy Data Extraction [ Geonhwa 20% / Taewoo 40% / Minju 40% ]
  - Using the recursive function `createNode()`, the component hierarchy data is extracted from the `fiber` object into a tree structure.
- Downloading the Tree Structure Data [ Geonhwa 40% / Taewoo 40% / Minju 20% ]
  - When the `Select Folder` button is clicked, a prompt appears asking for consent to download `data.json`, and the user's code is rendered in the Electron view (development mode rendering screen).
  - Then, `data.json` is immediately downloaded, the tree structure is rendered based on it, and `data.json` is deleted.
- Visualizing the Component Hierarchy [ Geonhwa 30% / Taewoo 20% / Minju 50% ]
  - In the tree structure, a `fiberNode` with a tag of 0 represents a `FunctionComponent`, which is displayed in blue.
  - Scrolling the mouse zooms in/out of the tree structure, and dragging moves the tree structure according to the cursor position.
  - Adjusting the WIDTH/HEIGHT slider bar at the top of the tree structure shrinks or enlarges the tree structure horizontally/vertically.
- Tree Structure Modal [ Geonhwa 20% / Taewoo 30% / Minju 50% ]
  - When hovering over a node in the tree structure, a modal appears showing component information.
  - The modal, which initially appears on the right side of the cursor, will switch to the left if the cursor is too close to the right edge, depending on the width of the Electron window and modal.
  - The modal shows the component's name, `props`, `local state`, and `redux state`.
- Code Viewer [ Geonhwa 40% / Taewoo 20% / Minju 40% ]
  - Clicking on a node in the tree displays the path and code of the JS file where the component is rendered.
  - If a non-component node is clicked or the X button is clicked, the code viewer and path information disappear.
- Folder Selection [ Geonhwa 30% / Taewoo 50% / Minju 20% ]
  - If the wrong folder is selected, an error popup and error page are rendered.
  - Clicking the folder selection button again allows the user to load a new project and render its tree structure.

<br>

# Timeline

Project Duration: March 6, 2023 (Mon) ~ March 30, 2023 (Thu)

- Week 1: Idea planning and mockup creation
- Week 2-3: Feature development
- Week 4: Writing test code, presentation

# Contacts

- Geonhwa Lee - ghlee2588@gmail.com
- Taewoo Kim - taewoo124@gmail.com
- Minju Park - mjuudev@gmail.com
