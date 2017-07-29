---
published: true
title: Using ARKit with Metal
author: <a href = "https://twitter.com/gpu3d" target="_blank">Marius Horga</a>
summary: <div><div style="display:inline-block;"><img src = "https://raw.githubusercontent.com/MetalKit/images/master/ARKit.jpg" alt="Metal 2" height="160" width="160"></div><div style="display:inline-block; width:75%; padding-left:1.5em; color:grey; vertical-align:middle;">Introducing ARKit - the newest framework from Apple. Defining the three stages in ARKit - Tracking, Scene Understanding and Rendering. Describing the main ARKit classes used - ARSession, ARSessionConfiguration, ARFrame, ARAnchor and ARCamera. Looking into tracking states and interruptions. Implementing a full Metal renderer for ARKit.</div></div>
layout: post
---
**Augmented Reality** provides a way of overlaying virtual content on top of real world views usually obtained from a mobile device camera. Last month at `WWDC 2017` we were all thrilled to see `Apple`'s new **ARKit** framework which is a high level `API` that works with `A9`-powered devices or newer, running on `iOS 11`. Some of the ARKit experiments we've already seen are outstanding, such as this one below: 

￼￼![alt text](https://github.com/MetalKit/images/blob/master/ARKit.gif?raw=true "ARKit")

There are three distinct layers in an `ARKit` application:

- **Tracking** - no external setup is necessary to do world tracking using visual inertial odometry.
- **Scene Understanding** - the ability of detecting scene attributes using plane detection, hit-testing and light estimation.
- **Rendering** - can be easily integrated because of the template `AR` views provided by `SpriteKit` and `SceneKit` but it can also be customized for `Metal`. All the pre-render processing is done by `ARKit` which is also responsible for image capturing using `AVFoundation` and `CoreMotion`.

In this first part of the series we will be looking mostly at `Rendering` in `Metal` and talk about the other two stages in the next part of this series. In an `AR` application, the `Tracking` and `Scene Understanding` are handled entirely by the `ARKit` framework while `Rendering` can be handled by either `SpriteKit`, `SceneKit` or `Metal`: 

￼￼![alt text](https://github.com/MetalKit/images/blob/master/ARKit1.png?raw=true "ARKit 1")

To get started, we need to have an **ARSession** instance that is set up by an **ARSessionConfiguration** object. Then, we call the __run()__ function on this configuration. The session also has __AVCaptureSession__ and __CMMotionManager__ objects running at the same time to get image and motion data for tracking. Finally, the session will output the current frame to an __ARFrame__ object:

![alt text](https://github.com/MetalKit/images/blob/master/ARKit2.png?raw=true "ARKit 2")

The `ARSessionConfiguration` object contains information about the type of tracking the session will have. The `ARSessionConfiguration` base configuration class provides __3__ degrees of freedom tracking (the device _orientation_) while its subclass, __ARWorldTrackingSessionConfiguration__, provides __6__ degrees of freedom tracking (the device _position_ and _orientation_). 

![alt text](https://github.com/MetalKit/images/blob/master/ARKit4.png?raw=true "ARKit 4")

When a device does not support world tracking, it falls back to the base  configuration:

{% highlight swift %}if ARWorldTrackingSessionConfiguration.isSupported { 
    configuration = ARWorldTrackingSessionConfiguration()
} else {
    configuration = ARSessionConfiguration() 
}
{% endhighlight %}

An `ARFrame` contains the captured image, tracking information and well as scene information via __ARAnchor__ objects that contain information about real world position and orientation and can be easily added, updated or removed from sessions. `Tracking` is the ability to determine the physical location in real time. The `World Tracking` however, determines both position and orientation, it works with physical distances, it’s relative to the starting position and provides `3D`-feature points. 

The last component of an `ARFrame` are __ARCamera__ objects which facilitate transforms (translation, rotation, scaling) and carry tracking state and camera intrinsics. The quality of tracking relies heavily on uninterrupted sensor data, static scenes and is more accurate when scenes have textured environment with plenty of complexity. Tracking state has three values: __Not Available__ (camera only has the identity matrix), __Limited__ (scene has insufficient features or is not static enough) and __Normal__ (camera is populated with data). Session interruptions are caused by camera input not being available or when tracking is stopped:

{% highlight swift %}func session(_ session: ARSession, cameraDidChangeTrackingState camera: ARCamera) { 
    if case .limited(let reason) = camera.trackingState {
        // Notify user of limited tracking state
    } 
}
func sessionWasInterrupted(_ session: ARSession) { 
    showOverlay()
}
func sessionInterruptionEnded(_ session: ARSession) { 
    hideOverlay()
    // Optionally restart experience
}
{% endhighlight %}

`Rendering` can be done in `SceneKit` using the `ARSCNView`'s delegate to add, update or remove nodes. Similarly, rendering can be done in `SpriteKit` using the `ARSKView` delegate which maps `SKNodes` to `ARAnchor` objects. Since `SpriteKit` is `2D`, it cannot use the real world camera position, so it projects the anchor positions into the `ARSKView` and then renders the sprite as a billboard (plane) at this projected location, so the sprite will always be facing the camera. For `Metal`, there is no customized `AR` view so that responsibility falls in programmer’s hands. For processing of rendered images we need to: 

- draw background camera image (generate a texture from the pixel buffer)
- update the virtual camera
- update the lighting
- update the transforms for geometry

All this information is in the `ARFrame` object. To access the frame, there are two options: _polling_ or using a _delegate_. We are going to describe the latter. I took the `ARKit` template for `Metal` and stripped it down to a minimum so I can better understand how it works. First thing I did was to remove all the `C` dependencies so bridging is not necessary anymore. It will be useful in the future to have it in place so types and enum constants can be shared between `API` code and shaders but for the purpose of this article it is not needed.

Next, on to __ViewController__ which will act as both our `MTKView` and `ARSession` delegates. We create a `Renderer` instance that will work with the delegates for real time updates to the application: 

{% highlight swift %}var session: ARSession!
var renderer: Renderer!

override func viewDidLoad() {
    super.viewDidLoad()
    session = ARSession()
    session.delegate = self
    if let view = self.view as? MTKView {
        view.device = MTLCreateSystemDefaultDevice()
        view.delegate = self
        renderer = Renderer(session: session, metalDevice: view.device!, renderDestination: view)
        renderer.drawRectResized(size: view.bounds.size)
    }
    let tapGesture = UITapGestureRecognizer(target: self, action: #selector(self.handleTap(gestureRecognize:)))
    view.addGestureRecognizer(tapGesture)
}
{% endhighlight %}

As you can see, we also added a gesture recognizer which we will use to add virtual content to our view. We first get the session’s current frame, then create a translation to put our object in front of the camera (__0.3__ meters in this case) and finally add a new anchor to our session using this transform:
    
{% highlight swift %}func handleTap(gestureRecognize: UITapGestureRecognizer) {
    if let currentFrame = session.currentFrame {
        var translation = matrix_identity_float4x4
        translation.columns.3.z = -0.3
        let transform = simd_mul(currentFrame.camera.transform, translation)
        let anchor = ARAnchor(transform: transform)
        session.add(anchor: anchor)
    }
}
{% endhighlight %}

We use the __viewWillAppear()__ and __viewWillDisappear()__ methods to start and pause the session:
    
{% highlight swift %}override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    let configuration = ARWorldTrackingSessionConfiguration()
    session.run(configuration)
}

override func viewWillDisappear(_ animated: Bool) {
    super.viewWillDisappear(animated)
    session.pause()
}
{% endhighlight %}

What’s left is only the delegate methods which we need to react to view updates or session errors and interruptions:
    
{% highlight swift %}func mtkView(_ view: MTKView, drawableSizeWillChange size: CGSize) {
    renderer.drawRectResized(size: size)
}

func draw(in view: MTKView) {
    renderer.update()
}

func session(_ session: ARSession, didFailWithError error: Error) {}

func sessionWasInterrupted(_ session: ARSession) {}

func sessionInterruptionEnded(_ session: ARSession) {}
{% endhighlight %}

Let's move to the __Renderer.swift__ file now. The first thing to notice is the use of a very handy protocol that will give us access to all the `MTKView` properties we need for the draw call later:

{% highlight swift %}protocol RenderDestinationProvider {
    var currentRenderPassDescriptor: MTLRenderPassDescriptor? { get }
    var currentDrawable: CAMetalDrawable? { get }
    var colorPixelFormat: MTLPixelFormat { get set }
    var depthStencilPixelFormat: MTLPixelFormat { get set }
    var sampleCount: Int { get set }
}
{% endhighlight %}

Now you can simply extend the `MTKView` class (in `ViewController`) so it conforms to this protocol:

{% highlight swift %}extension MTKView : RenderDestinationProvider {}
{% endhighlight %}

To have a high level view of the `Renderer` class, here is the pseudocode:

{% highlight swift %}init() {
    setupPipeline()
    setupAssets()
}
    
func update() {
    updateBufferStates()
    updateSharedUniforms()
    updateAnchors()
    updateCapturedImageTextures()
    updateImagePlane()
    drawCapturedImage()
    drawAnchorGeometry()
}
{% endhighlight %}
    
As always, we first setup the pipeline, here with the __setupPipeline()__ function. Then, in __setupAssets()__ we create our model which will be loaded every time we use our tap gesture recognizer. The `MTKView` delegate will call the __update()__ function for the needed updates and draw calls. Let's look at each of them in detail. First we have __updateBufferStates()__ which updates the locations we write to in our buffers for the current frame (we use a ring buffer with __3__ slots in this case):
       
{% highlight swift %}func updateBufferStates() {
    uniformBufferIndex = (uniformBufferIndex + 1) % maxBuffersInFlight
    sharedUniformBufferOffset = alignedSharedUniformSize * uniformBufferIndex
    anchorUniformBufferOffset = alignedInstanceUniformSize * uniformBufferIndex
    sharedUniformBufferAddress = sharedUniformBuffer.contents().advanced(by: sharedUniformBufferOffset)
    anchorUniformBufferAddress = anchorUniformBuffer.contents().advanced(by: anchorUniformBufferOffset)
}
{% endhighlight %}

Next, in __updateSharedUniforms()__ we update the shared uniforms of the frame and set up lighting for the scene:

{% highlight swift %}func updateSharedUniforms(frame: ARFrame) {
    let uniforms = sharedUniformBufferAddress.assumingMemoryBound(to: SharedUniforms.self)
    uniforms.pointee.viewMatrix = simd_inverse(frame.camera.transform)
    uniforms.pointee.projectionMatrix = frame.camera.projectionMatrix(withViewportSize: viewportSize, orientation: .landscapeRight, zNear: 0.001, zFar: 1000)
    var ambientIntensity: Float = 1.0
    if let lightEstimate = frame.lightEstimate {
        ambientIntensity = Float(lightEstimate.ambientIntensity) / 1000.0
    }
    let ambientLightColor: vector_float3 = vector3(0.5, 0.5, 0.5)
    uniforms.pointee.ambientLightColor = ambientLightColor * ambientIntensity
    var directionalLightDirection : vector_float3 = vector3(0.0, 0.0, -1.0)
    directionalLightDirection = simd_normalize(directionalLightDirection)
    uniforms.pointee.directionalLightDirection = directionalLightDirection
    let directionalLightColor: vector_float3 = vector3(0.6, 0.6, 0.6)
    uniforms.pointee.directionalLightColor = directionalLightColor * ambientIntensity
    uniforms.pointee.materialShininess = 30
}
{% endhighlight %}

Next, in __updateAnchors()__ we update the anchor uniform buffer with transforms of the current frame's anchors:

{% highlight swift %}func updateAnchors(frame: ARFrame) {
    anchorInstanceCount = min(frame.anchors.count, maxAnchorInstanceCount)
    var anchorOffset: Int = 0
    if anchorInstanceCount == maxAnchorInstanceCount {
        anchorOffset = max(frame.anchors.count - maxAnchorInstanceCount, 0)
    }
    for index in 0..<anchorInstanceCount {
        let anchor = frame.anchors[index + anchorOffset]
        var coordinateSpaceTransform = matrix_identity_float4x4
        coordinateSpaceTransform.columns.2.z = -1.0
        let modelMatrix = simd_mul(anchor.transform, coordinateSpaceTransform)
        let anchorUniforms = anchorUniformBufferAddress.assumingMemoryBound(to: InstanceUniforms.self).advanced(by: index)
        anchorUniforms.pointee.modelMatrix = modelMatrix
    }
}
{% endhighlight %}

Next, in __updateCapturedImageTextures()__ we create two textures from the provided frame's captured image:

{% highlight swift %}func updateCapturedImageTextures(frame: ARFrame) {
    let pixelBuffer = frame.capturedImage
    if (CVPixelBufferGetPlaneCount(pixelBuffer) < 2) { return }
    capturedImageTextureY = createTexture(fromPixelBuffer: pixelBuffer, pixelFormat:.r8Unorm, planeIndex:0)!
    capturedImageTextureCbCr = createTexture(fromPixelBuffer: pixelBuffer, pixelFormat:.rg8Unorm, planeIndex:1)!
}
{% endhighlight %}

Next, in __updateImagePlane()__ we update the texture coordinates of our image plane to aspect fill the viewport:
        
{% highlight swift %}func updateImagePlane(frame: ARFrame) {
    let displayToCameraTransform = frame.displayTransform(withViewportSize: viewportSize, orientation: .landscapeRight).inverted()
    let vertexData = imagePlaneVertexBuffer.contents().assumingMemoryBound(to: Float.self)
    for index in 0...3 {
        let textureCoordIndex = 4 * index + 2
        let textureCoord = CGPoint(x: CGFloat(planeVertexData[textureCoordIndex]), y: CGFloat(planeVertexData[textureCoordIndex + 1]))
        let transformedCoord = textureCoord.applying(displayToCameraTransform)
        vertexData[textureCoordIndex] = Float(transformedCoord.x)
        vertexData[textureCoordIndex + 1] = Float(transformedCoord.y)
    }
}
{% endhighlight %}

Next, in __drawCapturedImage()__ we draw the camera feed in the scene:

{% highlight swift %}func drawCapturedImage(renderEncoder: MTLRenderCommandEncoder) {
    guard capturedImageTextureY != nil && capturedImageTextureCbCr != nil else { return }
    renderEncoder.pushDebugGroup("DrawCapturedImage")
    renderEncoder.setCullMode(.none)
    renderEncoder.setRenderPipelineState(capturedImagePipelineState)
    renderEncoder.setDepthStencilState(capturedImageDepthState)
    renderEncoder.setVertexBuffer(imagePlaneVertexBuffer, offset: 0, index: 0)
    renderEncoder.setFragmentTexture(capturedImageTextureY, index: 1)
    renderEncoder.setFragmentTexture(capturedImageTextureCbCr, index: 2)
    renderEncoder.drawPrimitives(type: .triangleStrip, vertexStart: 0, vertexCount: 4)
    renderEncoder.popDebugGroup()
}
{% endhighlight %}
    
Finally, in __drawAnchorGeometry()__ we draw the anchors for the virtual content we created:

{% highlight swift %}func drawAnchorGeometry(renderEncoder: MTLRenderCommandEncoder) {
    guard anchorInstanceCount > 0 else { return }
    renderEncoder.pushDebugGroup("DrawAnchors")
    renderEncoder.setCullMode(.back)
    renderEncoder.setRenderPipelineState(anchorPipelineState)
    renderEncoder.setDepthStencilState(anchorDepthState)
    renderEncoder.setVertexBuffer(anchorUniformBuffer, offset: anchorUniformBufferOffset, index: 2)
    renderEncoder.setVertexBuffer(sharedUniformBuffer, offset: sharedUniformBufferOffset, index: 3)
    renderEncoder.setFragmentBuffer(sharedUniformBuffer, offset: sharedUniformBufferOffset, index: 3)
    for bufferIndex in 0..<mesh.vertexBuffers.count {
        let vertexBuffer = mesh.vertexBuffers[bufferIndex]
        renderEncoder.setVertexBuffer(vertexBuffer.buffer, offset: vertexBuffer.offset, index:bufferIndex)
    }
    for submesh in mesh.submeshes {
        renderEncoder.drawIndexedPrimitives(type: submesh.primitiveType, indexCount: submesh.indexCount, indexType: submesh.indexType, indexBuffer: submesh.indexBuffer.buffer, indexBufferOffset: submesh.indexBuffer.offset, instanceCount: anchorInstanceCount)
    }
    renderEncoder.popDebugGroup()
}
{% endhighlight %}

Back to the __setupPipeline()__ function which we briefly mentioned earlier. We create two render pipeline state objects, one for the captured image (the camera feed) and one for the anchors we create when placing virtual objects in the scene. As expected, each of the state objects will have their own pair of vertex and fragment functions - which brings us to the last file we need to look at - the __Shaders.metal__ file. In the first pair of shaders for the captured image, we pass through the image vertex's position and texture coordinate in the vertex shader:

{% highlight swift %}vertex ImageColorInOut capturedImageVertexTransform(ImageVertex in [[stage_in]]) {
    ImageColorInOut out;
    out.position = float4(in.position, 0.0, 1.0);
    out.texCoord = in.texCoord;
    return out;
}
{% endhighlight %}

In the fragment shader we sample the two textures to get the color at the given texture coordinate after which we return the converted `RGB` color:

{% highlight swift %}fragment float4 capturedImageFragmentShader(ImageColorInOut in [[stage_in]],
                                            texture2d<float, access::sample> textureY [[ texture(1) ]],
                                            texture2d<float, access::sample> textureCbCr [[ texture(2) ]]) {
    constexpr sampler colorSampler(mip_filter::linear, mag_filter::linear, min_filter::linear);
    const float4x4 ycbcrToRGBTransform = float4x4(float4(+1.0000f, +1.0000f, +1.0000f, +0.0000f),
                                                  float4(+0.0000f, -0.3441f, +1.7720f, +0.0000f),
                                                  float4(+1.4020f, -0.7141f, +0.0000f, +0.0000f),
                                                  float4(-0.7010f, +0.5291f, -0.8860f, +1.0000f));
    float4 ycbcr = float4(textureY.sample(colorSampler, in.texCoord).r, textureCbCr.sample(colorSampler, in.texCoord).rg, 1.0);
    return ycbcrToRGBTransform * ycbcr;
}
{% endhighlight %}

In the second pair of shaders for the anchor geometry, in the vertex shader we calculate the position of our vertex in clip space and output for clipping and rasterization, then color each face a different color, then calculate the positon of our vertex in eye space and finally rotate our normals to world coordinates:

{% highlight swift %}vertex ColorInOut anchorGeometryVertexTransform(Vertex in [[stage_in]],
                                                constant SharedUniforms &sharedUniforms [[ buffer(3) ]],
                                                constant InstanceUniforms *instanceUniforms [[ buffer(2) ]],
                                                ushort vid [[vertex_id]],
                                                ushort iid [[instance_id]]) {
    ColorInOut out;
    float4 position = float4(in.position, 1.0);
    float4x4 modelMatrix = instanceUniforms[iid].modelMatrix;
    float4x4 modelViewMatrix = sharedUniforms.viewMatrix * modelMatrix;
    out.position = sharedUniforms.projectionMatrix * modelViewMatrix * position;
    ushort colorID = vid / 4 % 6;
    out.color = colorID == 0 ? float4(0.0, 1.0, 0.0, 1.0)  // Right face
              : colorID == 1 ? float4(1.0, 0.0, 0.0, 1.0)  // Left face
              : colorID == 2 ? float4(0.0, 0.0, 1.0, 1.0)  // Top face
              : colorID == 3 ? float4(1.0, 0.5, 0.0, 1.0)  // Bottom face
              : colorID == 4 ? float4(1.0, 1.0, 0.0, 1.0)  // Back face
              :                float4(1.0, 1.0, 1.0, 1.0); // Front face
    out.eyePosition = half3((modelViewMatrix * position).xyz);
    float4 normal = modelMatrix * float4(in.normal.x, in.normal.y, in.normal.z, 0.0f);
    out.normal = normalize(half3(normal.xyz));
    return out;
}
{% endhighlight %}

In the fragment shader, we calculate the contribution of the directional light as a sum of diffuse and specular terms, then we compute the final color by multiplying the sample from the color maps by the fragment's lighting value and finally use the color we just computed and the alpha channel of the color map for this fragment's alpha value:

{% highlight swift %}fragment float4 anchorGeometryFragmentLighting(ColorInOut in [[stage_in]],
                                               constant SharedUniforms &uniforms [[ buffer(3) ]]) {
    float3 normal = float3(in.normal);
    float3 directionalContribution = float3(0);
    {
        float nDotL = saturate(dot(normal, -uniforms.directionalLightDirection));
        float3 diffuseTerm = uniforms.directionalLightColor * nDotL;
        float3 halfwayVector = normalize(-uniforms.directionalLightDirection - float3(in.eyePosition));
        float reflectionAngle = saturate(dot(normal, halfwayVector));
        float specularIntensity = saturate(powr(reflectionAngle, uniforms.materialShininess));
        float3 specularTerm = uniforms.directionalLightColor * specularIntensity;
        directionalContribution = diffuseTerm + specularTerm;
    }
    float3 ambientContribution = uniforms.ambientLightColor;
    float3 lightContributions = ambientContribution + directionalContribution;
    float3 color = in.color.rgb * lightContributions;
    return float4(color, in.color.w);
}
{% endhighlight %}

If you run the app, you should be able to tap on the screen to add cubes on top of your live camera view, and move away or closer or around the cubes to see their different colors on each face, like this:

￼￼![alt text](https://github.com/MetalKit/images/blob/master/ARKit1.gif?raw=true "ARKit 1")

In the next part of the series we will look more into `Tracking` and `Scene Understanding` and see how plane detection, hit-testing, collisions and physics can make our experience even greater. The [source code](https://github.com/MetalKit/metal) is posted on `Github` as usual.
 
Until next time!
