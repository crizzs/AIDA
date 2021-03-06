<template lang="html">
  <v-tooltip right open-delay=700>
    <template v-slot:activator="{ on }">
      <v-btn
        id="tool"
        v-on="on"
        block
        text
        @click.native="initialiseTool"
      >
        <v-icon small>
          mdi-brush
        </v-icon>
      </v-btn>
    </template>
    <span> Paint Tool </span>
  </v-tooltip>
</template>

<script>
import paper from 'paper'
import { mapActions, mapState, mapGetters } from 'vuex'

export default {
  data () {
    return {
      toolPaint: null,
      strokeWidth: 2, // Default value, will be updated relative to view
      radius: 2000,
      brush: null,
      overlap: null,
      path: null,
      combinedPath: null
    }
  },

  computed: {
    ...mapState({
      viewportZoom: state => state.image.OSDviewer.viewport.getZoom(true),
      imageWidth: state =>
        state.image.OSDviewer.world.getItemAt(0).getContentSize().x,
      activeStep: state => state.app.activeStep
    })
  },

  created () {
    const toolMove = event => {
      this.brush = new paper.Path.Circle({
        radius: this.radius,
        position: event.point,
        strokeWidth: this.strokeWidth,
        strokeColor: new paper.Color({
          hue: 220,
          saturation: 0.7,
          lightness: 0.5,
          alpha: 1
        })
      })

      this.brush.removeOn({
        move: true,
        drag: true
      })
    }

    const toolDrag = event => {
      this.path = new paper.Path.Circle({
        radius: this.radius,
        position: event.point,
        strokeWidth: this.strokeWidth,
        strokeColor: this.getColor().stroke,
        fillColor: this.getColor().fill
      }).removeOn({
        drag: true,
        up: true,
        move: true
      })

      const overlap = paper.project.getItem({
        class: 'Path',
        match: function (item) {
          if (item.data.class !== 'megakaryocyte') { return true }
        },
        overlapping: this.path.bounds
      })

      if (overlap) {
        if (event.modifiers.alt) {
          this.combinedPath = overlap.subtract(this.path)
        } else {
          this.combinedPath = overlap.unite(this.path)
        }
        overlap.remove()
      }
    }

    const toolUp = event => {
      // Flag the annotation has been edited and the changes are not saved
      this.flagAnnotationEdits()
    }

    // Add the defined functions to the tool object.
    this.toolPaint = new paper.Tool()
    this.toolPaint.onMouseMove = toolMove
    this.toolPaint.onMouseDrag = toolDrag
    this.toolPaint.onMouseUp = toolUp
  },

  methods: {
    ...mapActions({
      prepareCanvas: 'annotation/prepareCanvas',
      flagAnnotationEdits: 'annotation/flagAnnotationEdits'
    }),

    ...mapGetters({
      getColor: 'annotation/getColor'
    }),

    initialiseTool () {
      // Prepare PaperJS canvas for interaction.
      this.prepareCanvas()

      // Activate the paperJS tool.
      this.toolPaint.activate()

      // Set the default radius relative to image size and zoom level.
      this.radius = this.imageWidth / (this.viewportZoom * 100)
      this.strokeWidth = this.imageWidth / (this.viewportZoom * 500)
    },

    // Helper function - calculate distance between 2 points:
    calculateDistance (firstPoint, secondPoint) {
      const x1 = firstPoint.x
      const y1 = firstPoint.y
      const x2 = secondPoint.x
      const y2 = secondPoint.y

      const distance = Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2))
      return distance
    }
  }
}
</script>
