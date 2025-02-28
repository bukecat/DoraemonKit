<template>
  <div>
    <div class="real-time-switch">
      <div>实时更新</div>
      <input class="switch" type="checkbox" v-model="isRealTime" />
    </div>
    <ElementTree :node="node" />
  </div>
</template>

<script>
import ElementTree from "./elementTree.vue";
import MutationObserver from "mutation-observer";
import { guid, $bus } from "../../assets/util";
export default {
  components: {
    ElementTree,
  },
  data() {
    return {
      node: null,
      observer: null,
      key: 0,
      isRealTime: null,
      config: {
        attributes: true,
        childList: true,
        characterData: true,
        subtree: true,
      },
    };
  },
  mounted() {
    this.node = this.getNode(document.documentElement);
    this.observer = new MutationObserver((mutations) => {
      for (let i = 0; i < mutations.length; i++) {
        let mutation = mutations[i];
        if (this._isInDokit(mutation.target)) {
          continue;
        }
        this.onMutation(mutation);
      }
    });
    this.isRealTime = JSON.parse(localStorage.getItem("dokitElementRealTime")) ?? true;
  },
  destroyed() {
    this.observer.disconnect();
  },
  watch: {
    isRealTime: {
      handler: function (newval) {
        if (newval) {
          this.node = this.getNode(document.documentElement);
          $bus.emit(this.node.key);
          localStorage.setItem("dokitElementRealTime", newval);
          this.observer.observe(document.documentElement, this.config);
        } else if(newval===false){
          this.observer.disconnect();
          localStorage.setItem("dokitElementRealTime", newval);
        }
      },
      immediate: true,
    },
  },
  methods: {
    getNode(elem) {
      if (this._isIgnoredElement(elem)) {
        return undefined;
      }

      let node = elem.__dokitForWeb_node || {};

      // basic node info
      node.nodeType = elem.nodeType;
      node.key = node.key || guid();
      node.nodeName = elem.nodeName;
      node.tagName = elem.tagName || "";
      node.textContent = "";
      if (
        node.nodeType == elem.TEXT_NODE ||
        node.nodeType == elem.DOCUMENT_TYPE_NODE
      ) {
        node.textContent = elem.textContent;
      }

      // attrs
      node.id = elem.id || "";
      node.className = elem.className || "";
      node.attributes = [];
      if (elem.hasAttributes && elem.hasAttributes()) {
        for (let i = 0; i < elem.attributes.length; i++) {
          node.attributes.push({
            name: elem.attributes[i].name,
            value: elem.attributes[i].value || "",
          });
        }
      }

      // child nodes
      node.childNodes = [];
      if (elem.childNodes.length > 0) {
        for (let i = 0; i < elem.childNodes.length; i++) {
          let child = this.getNode(elem.childNodes[i]);
          if (!child) {
            continue;
          }
          node.childNodes.push(child);
        }
      }

      // save node to element for further actions
      elem.__dokitForWeb_node = node;
      return node;
    },
    _isIgnoredElement(elem) {
      // empty or line-break text
      if (elem.nodeType == elem.TEXT_NODE) {
        if (
          elem.textContent.replace(
            /^[\s\uFEFF\xA0]+|[\s\uFEFF\xA0]+$|\n+/g,
            ""
          ) == ""
        ) {
          // trim
          return true;
        }
      }
      return false;
    },
    _isInDokit(elem) {
      let target = elem;
      while (target != undefined) {
        if (target.id == "dokit-root") {
          return true;
        }
        target = target.parentNode || undefined;
      }
      return false;
    },
    onMutation(mutation) {
      switch (mutation.type) {
        case "childList":
          if (mutation.removedNodes.length > 0) {
            this.onChildRemove(mutation);
          }
          if (mutation.addedNodes.length > 0) {
            this.onChildAdd(mutation);
          }
          break;
        case "attributes":
          this.onAttributesChange(mutation);
          break;
        case "characterData":
          this.onCharacterDataChange(mutation);
          break;
        default:
          break;
      }
    },
    onAttributesChange(mutation) {
      let node = mutation.target.__dokitForWeb_node;
      if (!node) {
        return;
      }
      node = this.getNode(mutation.target);
      $bus.emit(node.key + "refreshMy");
    },
    onCharacterDataChange(mutation) {
      let node = mutation.target.__dokitForWeb_node;
      if (!node) {
        return;
      }
      node = this.getNode(mutation.target);
      $bus.emit(node.key + "refreshMy");
    },
    onChildAdd(mutation) {
      let $parent = mutation.target,
        parentNode = $parent.__dokitForWeb_node;
      if (!parentNode) {
        return;
      }
      this.getNode($parent);
      $bus.emit(parentNode.key + "refreshChild");
    },
    onChildRemove(mutation) {
      let $parent = mutation.target,
        parentNode = $parent.__dokitForWeb_node;
      if (!parentNode) {
        return;
      }
      for (let i = 0; i < mutation.removedNodes.length; i++) {
        let $target = mutation.removedNodes[i],
          key = $target?.__dokitForWeb_node?.key;
        if (!key) {
          continue;
        }
        // remove view
        this.deleteNode(parentNode, key);
      }
      $bus.emit(parentNode.key + "refreshChild");
    },
    deleteNode(parentNode, nodeKey) {
      let deleteNodeIndex = parentNode.childNodes.findIndex(
        (item) => item.key === nodeKey
      );
      parentNode.childNodes.splice(deleteNodeIndex, 1);
    },
  },
};
</script>

<style lang="less" scope>
.real-time-switch {
  display: flex;
  flex-direction: row;
  align-items: center;
  font-size: 18px;
  padding: 5px;
  .switch {
    appearance: none;
    -moz-appearance: button;
    -webkit-appearance: none;
  }
  .switch {
    position: relative;
    margin: 0;
    width: 40px;
    height: 24px;
    border: 1px solid #ebebf9;
    outline: 0;
    border-radius: 16px;
    box-sizing: border-box;
    background-color: #ebebf9;
    -webkit-transition: background-color 0.1s, border 0.1s;
    transition: background-color 0.1s, border 0.1s;
    margin-left: auto;
    &:before {
      content: " ";
      position: absolute;
      top: 0;
      left: 0;
      width: 38px;
      height: 22px;
      border-radius: 19px;
      background-color: #ebebf9;
      -webkit-transition: -webkit-transform 0.35s cubic-bezier(0.45, 1, 0.4, 1);
      transition: -webkit-transform 0.35s cubic-bezier(0.45, 1, 0.4, 1);
      transition: transform 0.35s cubic-bezier(0.45, 1, 0.4, 1);
    }

    &:after {
      content: " ";
      position: absolute;
      top: 0;
      left: 1px;
      width: 22px;
      height: 22px;
      border-radius: 15px;
      background-color: #ffffff;
      /*box-shadow: 0 1PX 3PX rgba(0, 0, 0, 0.4);*/
      -webkit-transition: -webkit-transform 0.35s
        cubic-bezier(0.4, 0.4, 0.25, 1.35);
      transition: -webkit-transform 0.35s cubic-bezier(0.4, 0.4, 0.25, 1.35);
      transition: transform 0.35s cubic-bezier(0.4, 0.4, 0.25, 1.35);
    }

    &:checked {
      background: rgb(69,124,190);
      border: solid 1px rgb(69,124,190);
    }

    &:checked:before {
      transform: scale(0);
    }

    &:checked:after {
      transform: translateX(15px);
    }
  }
}
</style>