<template>
<div class="blockly-flex">
  <div class="blockly-div"></div>
  <div class="blockly-code" v-show="showCode">
    <pre><code class="language-python" v-html="codeFormatted"></code></pre>
  </div>
  <div class="blockly-code" v-show="showCode">
    <pre><code class="language-xml" v-html="xmlFormatted" ></code></pre>
  </div>
</div>

</template>

<script>
import Blockly from 'node-blockly/browser'

import Prism from 'prismjs'

import Vue from 'vue'

import xml2js from 'xml2js'

Prism.highlightAll()

// Expose Blockly global for easier debugging
window.Blockly = Blockly

// ignore xml element and block element
Vue.config.ignoredElements.push('mutation')
Vue.config.ignoredElements.push('arg')
Vue.config.ignoredElements.push('xml')
Vue.config.ignoredElements.push('block')
Vue.config.ignoredElements.push('variables')
Vue.config.ignoredElements.push('variable')
Vue.config.ignoredElements.push('field')

export default {
  name: 'VBlockly',
  props: {
    toolbox: {
      type: String
    },
    blocks: {
      type: String
    },
    stubs: {
      type: String
    },
    workspace: {
      type: String
    },
    showCode: {
      type: Boolean,
      default: false
    }
  },
  computed: {
    codeFormatted () {
      return Prism.highlight(this.code, Prism.languages.python, 'python')
    },
    xmlFormatted () {
      let formatted = Prism.highlight(this.xml, Prism.languages.xml, 'xml')
      return formatted
    }
  },
  mounted () {
    let blocklyDiv = this.$el.querySelector('.blockly-div')
    // first load  the blocks, TODO: await this...
    this.loadBlocks().then(json => {
      this.currentBlocks = json
      Blockly.defineBlocksWithJsonArray(json)
      this.loadStubs()
    })
    this.loadToolbox().then((toolbox) => {
      let options = {}
      if (toolbox) {
        this.$el.appendChild(toolbox)
        options.toolbox = toolbox
        this.currentToolbox = toolbox
      }

      this.currentWorkspace = Blockly.inject(
        blocklyDiv,
        options
      )

      // load the custom workspace if we have it
      this.loadWorkspace().then(xml => {
        if (xml) {
          this.$el.appendChild(xml)
          Blockly.Xml.domToWorkspace(xml, this.currentWorkspace)
        }
      })
      this.currentWorkspace.addChangeListener(this.workspaceChanged)
    })
    this.observeResizes()
  },
  data () {
    return {
      currentToolbox: null,
      currentWorkspace: null,
      currentBlocks: [],
      code: '',
      xml: '',
      currentWorkspaceJson: null,
      resizeObserver: null
    }
  },
  methods: {
    resize () {
      // workaround to fix collapse problem
      if (this.currentWorkspace) {
        Blockly.svgResize(this.currentWorkspace)
      }

      // setTimeout(() => {
      //   window.dispatchEvent(new Event('resize'))
      //   let blocks = this.currentWorkspace.getTopBlocks()
      //   blocks.map(block => block.setCollapsed(true))
      //   blocks.map(block => block.setCollapsed(false))
      // }, 1000)
    },
    observeResizes () {
      let observer = new ResizeObserver(this.resize)
      observer.observe(this.$el)
      this.resizeObserver = observer
    },
    loadToolbox () {
      if (!this.toolbox) {
        // return null after the promise
        return new Promise((resolve) => { resolve(null) })
      }
      let promise = fetch(this.toolbox)
        .then(resp => {
          return resp.text()
        })
        .then(text => {
          let parser = new DOMParser()
          let doc = parser.parseFromString(text, 'application/xml')
          // append the xml elemt to this document
          let toolbox = doc.firstChild
          return toolbox
        })
      return promise
    },
    loadBlocks () {
      if (!this.blocks) {
        return new Promise(resolve => resolve([]))
      }
      let promise = fetch(this.blocks)
        .then(resp => {
          return resp.json()
        })
      return promise
    },
    loadStubs () {
      if (!this.stubs) {
        return new Promise((resolve) => { resolve(null) })
      }
      let promise = fetch(this.stubs)
        .then(resp => {
          return resp.text()
        })
        .then(text => {
          // TODO: use src??
          // eslint-disable-next-line
            return eval(text)
        })
      return promise
    },
    loadWorkspace () {
      if (!this.workspace) {
        return new Promise((resolve) => { resolve(null) })
      }
      let promise = fetch(this.workspace)
        .then(resp => {
          return resp.text()
        })
        .then(text => {
          let parser = new DOMParser()
          let doc = parser.parseFromString(text, 'application/xml')
          // append the xml elemt to this document
          let toolbox = doc.firstChild
          return toolbox
        })
      return promise
    },
    workspaceChanged (event) {
      if (event.type === Blockly.Events.BLOCK_CHANGE) {
        this.$emit(event.type, {
          event,
          workspace: this.currentWorkspace,
          block: this.currentWorkspace.getBlockById(event.blockId)
        })
      }
      if (event.type === Blockly.Events.BLOCK_CREATE) {
        this.$emit(event.type, {
          event,
          workspace: this.currentWorkspace,
          block: this.currentWorkspace.getBlockById(event.blockId)
        })
      }
      if (event.type === Blockly.Events.BLOCK_DELETE) {
        this.$emit(event.type, {
          event,
          workspace: this.currentWorkspace
        })
      }

      if (this.stub) {
        this.code = Blockly.Python.workspaceToCode(this.currentWorkspace)
      }
      let dom = Blockly.Xml.workspaceToDom(this.currentWorkspace)
      let xml = Blockly.Xml.domToPrettyText(dom)
      // let xml = new XMLSerializer().serializeToString(dom)
      this.xml = xml
      // TODO: serializae xml to json
      xml2js.parseString(xml, (err, result) => {
        if (err) {
          // toss it up
          throw (err)
        }
        this.currentWorkspaceJson = result.xml
      })
    }
  }
}

</script>
<style>
/* Fix layout issues. This can't be scoped because edit  widgets are placed outside of the component */
.blocklyHtmlInput {
  margin: 2px 0 0 4px;
}

</style>
<style scoped>

code:after, code:before, kbd:after, kbd:before {
  content: "";
  letter-spacing: initial;

}
.blockly-flex {
  display: flex;
  height: 100%;
}
.blockly-div {
  flex: 2;
  height: 100%;
  height: 100%;
}
.blockly-code {
  margin:5px;
  flex: 1;
}

.blockly-toolbox {
  display: none;
}
</style>