# CHAPTER 1 – RASTER ENGINE CODE.ORG

// ============================================================================
// VOID ENGINE
// RASTERIZATION PIPELINE DOCUMENTATION
// ============================================================================
//
// OVERVIEW
// --------
// The raster renderer is a traditional software 3D renderer.
//
// Unlike ray tracing and ray marching, rasterization:
//
//     Projects vertices onto the screen
//     Builds polygons from projected vertices
//     Draws visible faces
//
// Pipeline:
//
//     World Space
//          |
//          v
//     Camera Transform
//          |
//          v
//      View Space
//          |
//          v
//      Projection
//          |
//          v
//      Screen Space
//          |
//          v
//     Face Culling
//          |
//          v
//     Depth Sorting
//          |
//          v
//         Draw
//
// ============================================================================
// WORLD REPRESENTATION
// ============================================================================
//
// The world consists of cubes.
//
// Example:
//
//          +------+
//         /      /|
//        +------+ |
//        |      | +
//        |      |/
//        +------+
//
// Each cube stores:
//
//     Position
//     Size
//     Color
//     Radius
//     Cached vertices
//
// Example:
//
//     {
//         x: 4,
//         y: 0,
//         z: 7,
//         size: 1
//     }
//
// ============================================================================
// CUBE GEOMETRY
// ============================================================================
//
// Cubes are defined using:
//
//     CUBE_VERTS
//
// and
//
//     CUBE_FACES
//
// Vertices:
//
//        7------+6
//       /|     /|
//      3------+2|
//      | |    | |
//      |4+----|-5
//      |/     |/
//      0------+1
//
// Faces:
//
//     Front
//     Back
//     Left
//     Right
//     Top
//     Bottom
//
// Every face stores:
//
//     Vertex indices
//     Face normal
//     Shade multiplier
//
// ============================================================================
// CAMERA SYSTEM
// ============================================================================
//
// The camera stores:
//
//     Position
//     Rotation
//
// Example:
//
//     camera.x
//     camera.y
//     camera.z
//
//     camera.rotX
//     camera.rotY
//
// Rotation:
//
//     rotY = yaw
//     rotX = pitch
//
// ============================================================================
// CAMERA CACHE
// ============================================================================
//
// updateCameraCache()
//
// Computes:
//
//     sin(rotX)
//     cos(rotX)
//
//     sin(rotY)
//     cos(rotY)
//
// Once per frame.
//
// This avoids recalculating expensive trig functions
// hundreds or thousands of times.
//
// ============================================================================
// WORLD → CAMERA TRANSFORM
// ============================================================================
//
// transformPoint()
//
// Converts a world-space position into camera-space.
//
// Example:
//
//     World:
//
//         Cube @ (10,0,5)
//
//         Camera @ (8,0,2)
//
//     Relative:
//
//         (2,0,3)
//
// Then camera rotation is applied.
//
// Result:
//
//     Camera-space coordinates
//
// In camera-space:
//
//     +X = right
//     +Y = up
//     +Z = forward
//
// ============================================================================
// PROJECTION
// ============================================================================
//
// projectPoint()
//
// Converts a 3D point into a 2D screen coordinate.
//
// Formula:
//
//     scale = focalLength / depth
//
// Objects farther away:
//
//     Smaller
//
// Objects closer:
//
//     Larger
//
// Example:
//
//          Far Cube
//             []
//
//
//
//     Near Cube
//        [      ]
//
// Returned:
//
//     screenX
//     screenY
//     depth
//
// ============================================================================
// FRUSTUM CULLING
// ============================================================================
//
// sphereInFrustum()
//
// Purpose:
//
//     Skip objects outside camera view.
//
// Every cube is approximated by a bounding sphere.
//
// Example:
//
//             Camera
//                |
//           _____|_____
//          /           \
//         /  Visible    \
//        /    Area       \
//
// Cubes outside this region
// are ignored completely.
//
// This saves transform and draw calls.
//
// ============================================================================
// CHUNK SYSTEM
// ============================================================================
//
// The world is divided into chunks.
//
// Example:
//
//     +----+----+----+
//     | A  | B  | C  |
//     +----+----+----+
//     | D  | E  | F  |
//     +----+----+----+
//
// Each cube belongs to a chunk.
//
// Instead of checking:
//
//     Every cube
//
// the renderer checks:
//
//     Nearby chunks
//
// This dramatically reduces work.
//
// ============================================================================
// VERTEX TRANSFORMATION
// ============================================================================
//
// For visible cubes:
//
//     All 8 vertices
//
// are transformed into camera-space.
//
// Example:
//
//     Cube
//
//        8 vertices
//
//     Transform
//
//        8 camera-space vertices
//
// These transformed vertices are reused
// for every face of the cube.
//
// ============================================================================
// BACKFACE CULLING
// ============================================================================
//
// Purpose:
//
//     Remove faces pointing away from camera.
//
// Example:
//
//       Camera
//          |
//          v
//
//       +------+
//       |      |
//       +------+
//
// Back side is invisible.
//
// The face normal is compared
// against the camera direction.
//
// If:
//
//     dot <= 0
//
// the face is skipped.
//
// ============================================================================
// SCREEN AREA CULLING
// ============================================================================
//
// Very small faces are skipped.
//
// Example:
//
//     Face occupies:
//
//         2 pixels
//
// Rendering it is usually pointless.
//
// If:
//
//     area < MIN_FACE_AREA
//
// the face is discarded.
//
// ============================================================================
// RENDER QUEUE
// ============================================================================
//
// Visible faces are stored in:
//
//     renderQueue[]
//
// Each entry contains:
//
//     Projected vertices
//     Color
//     Average depth
//
// Example:
//
//     {
//         p: [...],
//         c: "red",
//         z: 12.3
//     }
//
// ============================================================================
// DEPTH SORTING
// ============================================================================
//
// Faces are sorted:
//
//     Furthest → Nearest
//
// Example:
//
//     Face A = z 20
//     Face B = z 10
//
// Draw:
//
//     Face A first
//     Face B second
//
// This is known as:
//
//     Painter's Algorithm
//
// Because Game Lab has no z-buffer.
//
// ============================================================================
// SHADING
// ============================================================================
//
// Each face stores a brightness value.
//
// Example:
//
//     Front  = 1.0
//     Back   = 0.85
//     Side   = 0.70
//     Top    = 0.55
//     Bottom = 0.40
//
// These values create fake directional lighting.
//
// Example:
//
//       Bright
//         |
//         v
//
//       +------+
//      /      /|
//     +------+ |
//     |      | +
//     |      |/
//     +------+
//
// ============================================================================
// DRAWING
// ============================================================================
//
// Faces are drawn using:
//
//     quad()
//
// Example:
//
//     quad(
//         x1,y1,
//         x2,y2,
//         x3,y3,
//         x4,y4
//     );
//
// Because Game Lab lacks GPU acceleration,
// every quad is rendered on the CPU.
//
// ============================================================================
// GRID RENDERER
// ============================================================================
//
// The floor grid is a separate renderer.
//
// Process:
//
//     Transform line endpoints
//     Project endpoints
//     Draw line
//
// Grid lines are skipped if:
//
//     Too far away
//
// or
//
//     Outside render distance
//
// ============================================================================
// PERFORMANCE OPTIMIZATIONS
// ============================================================================
//
// The raster renderer uses:
//
//     Chunk culling
//     Distance culling
//     Frustum culling
//     Backface culling
//     Area culling
//     Cached trig values
//     Cached cube vertices
//     Face limits
//
// These optimizations make real-time
// software 3D rendering possible in Game Lab.
//
// ============================================================================
// SUMMARY
// ============================================================================
//
// RASTERIZATION
//
//     Transform vertices
//     Project vertices
//     Build polygons
//     Sort polygons
//     Draw polygons
//
// Advantages:
//
//     Fastest renderer
//     Precise cube geometry
//     Works well with large worlds
//
// Disadvantages:
//
//     Requires depth sorting
//     No true reflections
//     No true global illumination
//
// This renderer serves as the primary
// real-time rendering mode for VOID ENGINE.
//
// ============================================================================






# CHAPTER 2 – RAYTRACING ENGINE CODE.ORG
// ============================================================================
// RAY TRACING RENDERER
// ============================================================================
//
// OVERVIEW
// --------
// The ray tracer renders the scene by shooting a ray through every screen
// pixel and testing for intersections against every cube in the world.
//
// Pipeline:
//
//     Screen Pixel
//           |
//           v
//      Generate Ray
//           |
//           v
//      Rotate by Camera
//           |
//           v
//    Test Against Cubes
//           |
//           v
//     Find Closest Hit
//           |
//           v
//      Shade + Draw
//
// Unlike rasterization, the ray tracer does not project cube vertices.
// Instead, it directly asks:
//
//     "What object does this ray hit first?"
//
// ============================================================================
// STEP 1: GENERATE SCREEN RAY
// ============================================================================
//
// Every pixel is converted into normalized device coordinates:
//
//     ndcX = -1 ... +1
//     ndcY = -1 ... +1
//
// Example:
//
//     (-1,+1)       (+1,+1)
//          +-------+
//          |       |
//          |   •   |
//          |       |
//          +-------+
//     (-1,-1)       (+1,-1)
//
// These coordinates are converted into a camera-space ray.
//
// Example:
//
//     Camera
//        |
//        |
//        +----> Viewing Plane
//
// Each pixel receives a unique ray direction.
//
// ============================================================================
// STEP 2: CAMERA ROTATION
// ============================================================================
//
// Rays are rotated according to camera yaw and pitch.
//
// Yaw:
//
//     rx = rcx*cosY - rcz*sinY
//     rz = rcx*sinY + rcz*cosY
//
// Pitch:
//
//     ry = rcy*cosX - rz*sinX
//     rz = rcy*sinX + rz*cosX
//
// This transforms camera-space rays into world-space rays.
//
// ============================================================================
// STEP 3: NORMALIZATION
// ============================================================================
//
// vecNorm() converts the ray direction into a unit vector.
//
// Before:
//
//     [3,2,8]
//
// After:
//
//     [0.34,0.23,0.91]
//
// Length becomes exactly 1.
//
// This ensures distance calculations remain accurate.
//
// ============================================================================
// STEP 4: RAY / AABB INTERSECTION
// ============================================================================
//
// Every cube is represented as an AABB
// (Axis Aligned Bounding Box).
//
//          +------+
//         /      /|
//        +------+ |
//        |      | +
//        |      |/
//        +------+
//
// rayAABB() uses the slab intersection algorithm.
//
// For each axis:
//
//     X entry
//     X exit
//
//     Y entry
//     Y exit
//
//     Z entry
//     Z exit
//
// The overlapping interval is calculated.
//
// If:
//
//     tMin > tMax
//
// the ray missed.
//
// Otherwise:
//
//     tMin
//
// is the hit distance.
//
// ============================================================================
// STEP 5: CLOSEST HIT SEARCH
// ============================================================================
//
// Every cube is tested.
//
// Example:
//
//     Ray
//      |
//      v
//
//   Cube A   t=4
//   Cube B   t=9
//   Cube C   t=15
//
// Cube A is selected because it is closest.
//
// ============================================================================
// STEP 6: SHADING
// ============================================================================
//
// Once a hit is found:
//
//     hitPoint = origin + direction * distance
//
// The hit face is approximated by checking which cube boundary
// the hit point lies closest to.
//
// Different faces receive different brightness values.
//
// Example:
//
//     Top    = dark
//     Side   = medium
//     Front  = bright
//
// Optional fog:
//
//     fog = distance / renderDistance
//
// Final color:
//
//     objectColor * brightness
//           +
//     skyColor * fog
//
// ============================================================================
// STEP 7: DRAW
// ============================================================================
//
// Pixels are rendered as blocks:
//
//     rayResolution x rayResolution
//
// Example:
//
//     resolution = 6
//
//     +------+
//     |██████|
//     |██████|
//     |██████|
//     +------+
//
// Lower resolution dramatically improves performance.
//
// ============================================================================
// PERFORMANCE
// ============================================================================
//
// Complexity:
//
//     pixels × cubes
//
// Example:
//
//     66 × 66 pixels
//     30 cubes
//
// ≈ 130,000 cube intersection tests/frame
//
// This is expensive.
// ============================================================================






# CHAPTER 3 – RAYMARCHING ENGINE CODE.ORG
// ============================================================================
// RAY MARCHING RENDERER
// ============================================================================
//
// OVERVIEW
// --------
// The ray marcher renders the world using Signed Distance Functions (SDFs).
//
// Instead of asking:
//
//     "Did I hit something?"
//
// it asks:
//
//     "How far away is the nearest surface?"
//
// Pipeline:
//
//     Screen Pixel
//           |
//           v
//      Generate Ray
//           |
//           v
//       March Ray
//           |
//           v
//      Query Scene SDF
//           |
//           v
//      Move Forward
//           |
//           v
//      Hit / Miss
//           |
//           v
//      Shade + Draw
//
// ============================================================================
// SIGNED DISTANCE FUNCTIONS
// ============================================================================
//
// An SDF returns the distance from a point to the nearest surface.
//
// Example:
//
//          Cube
//       +--------+
//
// Point •
//
// Distance:
//
//     5.3 units
//
// The function returns:
//
//     5.3
//
// If inside:
//
//     negative distance
//
// If on surface:
//
//     0
//
// ============================================================================
// sdfBox()
// ============================================================================
//
// sdfBox() computes the distance to a cube.
//
// Input:
//
//     point position
//     cube position
//     cube size
//
// Output:
//
//     shortest distance to cube surface
//
// Example:
//
//     Point outside cube
//
//     distance = 3.2
//
//     Point touching cube
//
//     distance = 0
//
//     Point inside cube
//
//     distance < 0
//
// ============================================================================
// sceneSDF()
// ============================================================================
//
// sceneSDF() evaluates all cubes.
//
// Example:
//
//     Cube A = 5.2
//     Cube B = 1.8
//     Cube C = 8.1
//
// Result:
//
//     nearest = 1.8
//
// The nearest distance determines how far the ray can safely move.
//
// ============================================================================
// STEP 1: GENERATE RAY
// ============================================================================
//
// Same process as the ray tracer.
//
// Pixel
//    |
//    v
//
// Camera-space ray
//
//    |
//    v
//
// Rotated by camera
//
//    |
//    v
//
// Normalized direction
//
// ============================================================================
// STEP 2: BEGIN MARCHING
// ============================================================================
//
// Start:
//
//     t = 0
//
// Position:
//
//     camera position
//
// Example:
//
//     Camera
//       •
//
// ============================================================================
// STEP 3: SAMPLE SCENE
// ============================================================================
//
// Current sample:
//
//     P = origin + ray * t
//
// sceneSDF(P)
//
// returns:
//
//     distance to nearest object
//
// Example:
//
//     P
//      |
//      v
//
//     nearest surface = 7
//
// Result:
//
//     d = 7
//
// ============================================================================
// STEP 4: SPHERE TRACING
// ============================================================================
//
// The ray advances by the exact safe distance.
//
// Example:
//
//     distance = 7
//
//     move forward 7 units
//
//     distance = 3
//
//     move forward 3 units
//
//     distance = 1
//
//     move forward 1 unit
//
//     distance = 0.2
//
//     move forward 0.2 units
//
// This is:
//
//     t += distance
//
// The ray automatically slows as it approaches geometry.
//
// ============================================================================
// STEP 5: HIT DETECTION
// ============================================================================
//
// If:
//
//     distance < SURF
//
// we consider the surface hit.
//
// Example:
//
//     SURF = 0.05
//
//     distance = 0.02
//
// Hit detected.
//
// This prevents infinite precision loops.
//
// ============================================================================
// STEP 6: TERMINATION
// ============================================================================
//
// A ray stops when:
//
//     distance < SURF
//
// OR
//
//     t > MAX_DIST
//
// OR
//
//     steps > MAX_STEPS
//
// Example:
//
//     MAX_STEPS = 32
//
// If 32 marches occur without a hit,
// the ray is considered a miss.
//
// ============================================================================
// STEP 7: FAKE AMBIENT OCCLUSION
// ============================================================================
//
// Lighting is estimated using march count.
//
//     ao = 1 - steps/MAX_STEPS
//
// Fewer steps:
//
//     brighter
//
// More steps:
//
//     darker
//
// Example:
//
//     4 steps  -> bright
//     20 steps -> medium
//     30 steps -> dark
//
// This is not true AO,
// but often produces convincing depth.
//
// ============================================================================
// STEP 8: FOG
// ============================================================================
//
// Optional fog:
//
//     fog = distance/MAX_DIST
//
// Near:
//
//     object color
//
// Far:
//
//     sky color
//
// Creates atmospheric depth.
//
// ============================================================================
// STEP 9: DRAW
// ============================================================================
//
// Like ray tracing,
// pixels are rendered as blocks.
//
//     resolution = 6
//
//     ██████
//     ██████
//     ██████
//
// This greatly reduces the number of rays required.
//
// ============================================================================
// PERFORMANCE
// ============================================================================
//
// Complexity:
//
//     pixels × steps × cubes
//
// Example:
//
//     66 × 66 pixels
//     32 steps
//     30 cubes
//
// ≈ 4.2 million SDF evaluations/frame
//
// This is why ray marching is significantly slower than ray tracing.
//
// ============================================================================
// SUMMARY
// ============================================================================
//
// RAY TRACING
//
//     Shoot ray
//     Intersect geometry
//     Exact hit
//
// RAY MARCHING
//
//     Shoot ray
//     Query distance field
//     Walk toward geometry
//
// Ray tracing:
//
//     "Did I hit something?"
//
// Ray marching:
//
//     "How far am I from something?"
//
// ============================================================================


# WHAT CHANGES IN THE HTML
// ============================================================================
// SUMMARY: HTML + JS HYBRID VS PURE JS ENGINE
// ============================================================================
//
// Code.org: 
//
//     Everything exists in JavaScript
//     No DOM structure
//     No buttons, no menus in HTML
//     Rendering + UI + input all in code
//
//     You would manually create:
//         canvas via JS
//         HUD via JS
//         sliders via JS
//         event listeners via JS
//
//     Example flow:
//         init()
//         createUI()
//         createCanvas()
//         startLoop()
//
// HTML: 
//
//     HTML defines structure
//     JS drives logic
//
//     HTML provides:
//         canvas element
//         menu layout
//         HUD text slots
//         sliders + buttons
//
//     JS provides:
//         rendering system
//         camera system
//         world simulation
//         input handling
//         state management
//
// ============================================================================
// KEY DIFFERENCE: WHO OWNS THE UI
// ============================================================================
//
// Code.org:
//
//     UI is CREATED by JavaScript
//     UI is UPDATED by JavaScript
//     UI is STORED in JavaScript objects
//
// HTML:
//
//     UI is CREATED by HTML
//     UI is CONTROLLED by JavaScript
//     JS only MUTATES existing DOM nodes
//
// ============================================================================
// KEY DIFFERENCE: ENGINE BOOTSTRAP
// ============================================================================
//
// Code.org:
//
//     Everything starts in JS:
//
//         function boot() {
//             createCanvas()
//             buildHUD()
//             initWorld()
//             startLoop()
//         }
//
// HTML:
//
//     HTML already exists when JS starts:
//
//         <canvas id="canvas"></canvas>
//         <div id="hud"></div>
//
//     JS only attaches behavior:
//
//         getElementById(...)
//         modify text / styles
//         bind events
//
// ============================================================================
// KEY DIFFERENCE: RENDERING PIPELINE
// ============================================================================
//
// Code.org:
//
//     JS owns full lifecycle:
//
//         create canvas
//         attach to document
//         render loop controls everything
//
// HTML:
//
//     HTML owns canvas element
//     JS only draws into it:
//
//         ctx = canvas.getContext("2d")
//
//     JS never creates the canvas,
//     only uses it.
//
// ============================================================================
// KEY DIFFERENCE: INPUT SYSTEM
// ============================================================================
//
// Code.org:
//
//     JS builds input layer manually:
//         window.addEventListener(...)
//         custom key state system
//         UI buttons created in JS
//
// HTML:
//
//     HTML defines input nodes:
//         <button onclick="...">
//         <input oninput="...">
//
//     JS only implements the logic behind them.
//
// ============================================================================
// KEY DIFFERENCE: ARCHITECTURE STYLE
// ============================================================================
//
// Code.org:
//
//     "ENGINE-FIRST"
//
//     JS is the world
//     JS is the UI
//     JS is the renderer
//
// HTML:
//
//     "DOM-FIRST + ENGINE BACKEND"
//
//     HTML = skeleton
//     JS = brain
//     Canvas = rendering surface
//
// ============================================================================
// FINAL MODEL
// ============================================================================
//
// Code.org:
//
//     JavaScript builds EVERYTHING dynamically
//     No external structure
//     Fully procedural engine boot
//
// HTML:
//
//     HTML builds structure
//     JS injects behavior + simulation
//     Canvas is the only fully JS-driven surface
//
// ============================================================================
// SIMPLE ONE-LINE DIFFERENCE
// ============================================================================
//
// Code.org:
//     "JS creates the world"
//
// HTML:
//     "HTML defines the world shell, JS animates it"
//
// ============================================================================ 




