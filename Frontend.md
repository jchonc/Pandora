# Rewrite the RL6 expression editor in React

## Background & History

I am not very happy with the existing RL6 expression editor. The new rewrite did take in a lot of new design ideas to be fair, but the 6K lines of vanilla JavaScript code is a can of worm nobody wanted to touch -- especially after the main developer left.

## Why re-write, why react

When I was reading through the code, I cannot help but noticed we have spent a lot of code manipulate the two trees and keep them in-sync, the data tree in memory and GUI tree in DOM. Thus the original idea of rewriting it in modern JavaScript framework will cut the workload in half since the manipulation of DOM should be handled by the framework.

Given our past (not so sweet) experience with EmberJS we are trying ReactJS, thus the experiment.

Beside the workload issue, the existing code did not use some of the mordern CSS features. (pseudo selector like before/after/first/last)

## Code highlight

The code could be found [here](https://github.com/jchonc/react-expr-editor/tree/ant-mobx-tree)

* The main structure is from an ejected Create-React-App folder, using TypeScript. I ejected it because of the need of SCSS. But with the configless [WebPack4 release](https://medium.com/webpack/webpack-4-released-today-6cdb994702d4) this need might be gone.
* Mobx as state managment. The original branch is using the local state since this is supposed to a component. The recurrsive nature of the logic expression tree has made it less attractive to use redux/store. Thus we settled with mobx and going back to the OO route, [pushing actions with their data together](https://github.com/jchonc/react-expr-editor/blob/ant-mobx-tree/src/types/AbstractNode.ts). Mobx also provides a nice (more elegant IMHO) way for observable/observsor - similar to EmberJS.
* Jest and Enzyme based tests. We have gone with the main stream of unit testing. So far we have found it workable, but when touching the fine interactive UI elements it's quite [tedious](https://github.com/jchonc/react-expr-editor/blob/ant-mobx-tree/src/components/editors/ExpressionValueDate.test.tsx) and often very tightly coupled with the components.
* Express based fake API server. One of the features we missed the most from Ember. Since we do have NodeJS as the back end we added [a server mock up](https://github.com/jchonc/react-expr-editor/blob/ant-mobx-tree/server/app.js).

## GUI

We are dying to find a good design/component sets on this new platform. Bootstrap didn't cover all the componentry. While we haven't fully checked the DevExtreme/KendoUI out I bumped into [a framework from Alibaba](https://ant.design/docs/react/introduce). It seems they have encapsulated both the layout and controls. Obviously this library is written for their own [Ant Finance](https://www.antfin.com/) application.

## Drag and drop & Validation

Jacob has added the magic of drag-n-drop to the code thus he might know better.
Validation is also his work. So we might want to ask him to add more details here. 

## Missing features

* XML/JSON conversion. This editor was written for the patient experience piece, thus it takes the expression in JSON format. And it assumes there could be multiple root tables(entities) in each module.

* It did not implement the system variables yet.
* I think the way it [handles the ajax/loading](https://github.com/jchonc/react-expr-editor/blob/ant-mobx-tree/src/stores/UtilityStore.ts) could be more elegant.

## Use them in RL6

Using this library in RL6 involves a hack to expose uglified objects to the outside world. Here are the steps I have tried.

* Skip the initial rendering and expose the needed classes globally. 
![Screenshot of changes](https://github.com/jchonc/Pandora//raw/master/mainExport.png)
* Reference the script and other assets in your page and take the parameters and original rendering in your own code.
![Screenshot of hydration](https://github.com/jchonc/Pandora//raw/master/hydration.png)

