<template>
  <div>
    <!-- Video Elements for before and after streams -->
    <video ref="beforeVideo" autoplay playsinline></video>
    <video ref="afterVideo" autoplay playsinline></video>

    <!-- Image Elements for before and after waveform -->
    <img ref="waveformBefore" />
    <img ref="waveformAfter" />
  </div>
</template>

<script>
export default {
  data() {
    return {
      ws: null, // WebSocket connection
      contentType: '', // Store the content type for video or image
    };
  },
  mounted() {
    // Initialize WebSocket connection when component is mounted
    console.log("Vue component mounted. Setting up WebSocket connection...");
    this.ws = new WebSocket("ws://localhost:3000");
    this.ws.binaryType = "arraybuffer"; // Ensure binary data is handled as ArrayBuffer

    // Set up WebSocket event handlers
    this.ws.onopen = () => {
      console.log("WebSocket connection established.");
    };

    this.ws.onmessage = this.handleMessage;
    this.ws.onerror = this.handleError;
    this.ws.onclose = this.handleClose;
  },
  methods: {
    // Handle incoming WebSocket messages
    handleMessage(event) {
      try {
        const message = event.data;

        // If message is a string (likely JSON message)
        if (typeof message === 'string') {
          const parsedMessage = JSON.parse(message);
          console.log("Received JSON message:", parsedMessage);

          if (parsedMessage.type === "metadata") {
            // Handle metadata (video type)
            this.contentType = parsedMessage.contentType || "video/webm"; // Default to "video/webm"
          } else if (parsedMessage.type === "info") {
            // Handle info message (server is ready to send video)
            console.log("Info message received:", parsedMessage.message);
          } else {
            console.error("Unexpected JSON message:", parsedMessage);
          }

        } else if (message instanceof ArrayBuffer) {
          // If the message is binary (video data or waveform image)
          this.handleBinaryData(message);
        } else {
          console.error("Received unsupported data type:", message);
        }
      } catch (error) {
        console.error("Error handling WebSocket message:", error);
      }
    },

    // Handle binary data (video stream or waveform image)
    handleBinaryData(message) {
      if (!this.contentType) {
        console.warn("No contentType set. Unable to process binary data.");
        
        // Attempt to guess the type based on the data size or content (optional)
        if (message instanceof Blob) {
          // Check if Blob type is empty and set a default based on size or other heuristics
          if (!message.type) {
            console.log("Blob type is empty, setting default type.");
            this.contentType = "video/webm"; // Default to a reasonable MIME type
          } else {
            this.contentType = message.type; // Use the Blob's type if it's set
          }
        } else {
          console.error("Unsupported data type:", typeof message);
        }
      }

      if (this.contentType.startsWith("video/")) {
        // It's a video stream (WebM format)
        this.updateVideoStream(this.$refs.beforeVideo, message);
      } else if (this.contentType === "image/png") {
        // It's a waveform image (PNG format)
        this.updateWaveformImage(this.$refs.waveformBefore, message);
      } else {
        console.error("Unsupported content type:", this.contentType);
      }
    },

    // Update the video stream by converting the ArrayBuffer to a Blob
    updateVideoStream(videoElement, buffer) {
      try {
        if (!this.contentType || !this.contentType.startsWith("video/")) {
          console.error("Content type is not set or invalid for video stream.");
          return;
        }

        const videoBlob = new Blob([buffer], { type: this.contentType });
        const videoUrl = URL.createObjectURL(videoBlob);
        videoElement.src = videoUrl;

        // Add event listeners for video loading events
        videoElement.onloadstart = () => console.log("Video started loading...");
        videoElement.oncanplay = () => console.log("Video is ready to play.");
        videoElement.onerror = (error) => console.error("Error loading video:", error);
      } catch (error) {
        console.error("Error updating video stream:", error);
      }
    },

    // Update waveform image by setting the source to a base64 image
    updateWaveformImage(imageElement, buffer) {
      try {
        if (!this.contentType || this.contentType !== "image/png") {
          console.error("Content type is not set or invalid for image.");
          return;
        }

        // The message is expected to be a Blob, convert it to a base64 data URL
        const blob = new Blob([buffer], { type: "image/png" });
        const reader = new FileReader();

        reader.onloadend = () => {
          const base64Image = reader.result; // base64 encoded string
          imageElement.src = base64Image; // Set the base64 image to the image element
        };

        reader.readAsDataURL(blob); // Read the image as base64 data
      } catch (error) {
        console.error("Error updating waveform image:", error);
      }
    },

    // Handle WebSocket errors
    handleError(event) {
      console.error("WebSocket error:", event);
    },

    // Handle WebSocket close event
    handleClose(event) {
      console.log("WebSocket connection closed:", event);
    },
  },
};
</script>

<style scoped>
/* Styling for video and image elements */
video {
  width: 100%;
  max-width: 1280px;
  margin-bottom: 20px;
}

img {
  max-width: 1280px;
  margin-top: 20px;
  display: block;
}
</style>
