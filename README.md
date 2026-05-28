# Snowboarding Coach AI

Working with [SnowboardingExplained](https://www.youtube.com/@SnowboardingExplained) to coach pro rider **Andrew Somers** and hundreds of other snowboarders to the pro level. The ultimate goal is an AI-powered video coach, but for now we have tools that help us learn how to snowboard.

- Built a frame-level video comparison tool to compare snowboarder form versus ideal trick execution
- Developed a fast, shared video scrubber using HTTP range requests and CloudFront CDN for instant seeking
- Created using TypeScript, Python, Three.js, PyTorch, AWS, React, MongoDB, Playwright, 4D-Humans, FFmpeg

**Live Demo:** [snowboarding-explained.com](https://snowboarding-explained.com)

---

## Tool 1: Frame-by-Frame Video Comparison Player

A web-based video analysis tool that lets coaches compare two snowboarders side-by-side with frame-accurate synchronization.

### Core Features

**Multi-Cell Grid Layout**
- Configurable grid (1x2, 2x2, 2x3, etc.) for comparing multiple videos
- Each cell can hold video, 3D mesh, or comment panel
- Cells can be linked (video + comment panel side-by-side)
- Zustand store manages cell state with spatial indexing for position-based queries

**Synchronized Playback Engine**
- Single RAF loop at 60fps as source of truth for all timing
- Global playback controls (play/pause/seek/speed) sync all video cells
- Independent mesh playback with separate timing per cell
- Loop boundary synchronization (all cells pause at end, seek to frame 0, resume together)
- Speed control: 0.125x, 0.25x, 0.5x, 0.75x, 1x, 1.5x, 2x
- Frame-by-frame stepping (forward/backward)

**Video Alignment**
- Line up two different videos at different points
- Relative scrubbing mode: scrubbing one video maintains offset with others
- Baseline capture when entering relative mode
- Each cell tracks its own playback time while staying in sync

**Timeline Comments**
- Add timestamped comments to any video
- Comments stored in MongoDB with videoId, timestamp, frameNumber
- "Pause at comments" mode: video auto-pauses when reaching comment timestamps
- Comment highlighting syncs with playback position
- Comment panel can be opened in adjacent grid cell

**Three.js 3D Mesh Viewer**
- Real-time 3D skeleton and mesh rendering (6890 SMPL vertices)
- Dual camera control modes: Orbit (spherical) and Arcball (quaternion)
- Independent mesh scrubber with own play/pause/frame controls
- Geometry reuse pattern: create BufferGeometry once, update position attribute each frame
- Camera presets: top, front, back, left, right
- Nametag labels floating above mesh

**Video Modes**
- Original video playback
- Overlay video (with pose skeleton rendered)
- 3D mesh view (Three.js)
- Toggle between modes per cell

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (React + Vite)                  │
│                                                             │
│  GridLayout ─── GridCell ─── VideoDisplay / MeshViewer      │
│       │              │                                      │
│       │              └── CellScrubber / MeshScrubber        │
│       │                                                     │
│  PlaybackEngine (RAF loop @ 60fps)                          │
│       │                                                     │
│       ├── Global playback time (video cells)                │
│       ├── Independent mesh playback times (per cell)        │
│       ├── Comment state (per video)                         │
│       └── Cell registration (video elements, mesh cells)    │
│                                                             │
│  GridStore (Zustand)                                        │
│       ├── Cell UI state (contentType, videoId, nametag)     │
│       ├── Cell spatial index (row, col for positioning)     │
│       └── Shared controls (camera preset)                   │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                  Backend API (Express.js)                   │
│                                                             │
│  /api/videos ─── List, upload, delete videos                │
│  /api/mesh-data/:videoId ─── Mesh frames (vertices, faces)  │
│  /api/comments ─── CRUD for timeline comments               │
│  /api/job-status/:videoId ─── Processing status polling     │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│              Pose Service (Python Flask + GPU)              │
│                                                             │
│  4D-Humans ─── 3D pose reconstruction (Transformer-based)   │
│  ViTDet ─── Vision Transformer person detection             │
│  PHALP ─── Temporal tracking and smoothing                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Frontend Implementation

### PlaybackEngine

Central timing controller running RAF loop at 60fps.

```typescript
class PlaybackEngine {
  private _playbackTime: number = 0;
  private _isPlaying: boolean = false;
  private _playbackSpeed: number = 1;
  private lastFrameTime: number = 0;
  private frameIntervalMs: number;  // 1000 / fps
  
  // Per-cell state for independent playback
  private meshPlaybackTimes: Map<string, number> = new Map();
  private cellPlaybackTimes: Map<string, number> = new Map();
  
  private startRafLoop(): void {
    const loop = (currentTime: number) => {
      // Delta time calculation for frame-rate independence
      const deltaTime = currentTime - this.lastFrameTime;
      this.lastFrameTime = currentTime;

      if (this._isPlaying && !this.syncPauseActive) {
        // Core timing math: advance by delta * speed
        this._playbackTime += deltaTime * this._playbackSpeed;
        this.handleLooping();
        this.checkForLoopSync();
      }

      this.checkCommentEvents();
      this.emitEvent({ type: 'frameUpdate' });
      this.rafId = requestAnimationFrame(loop);
    };
    this.rafId = requestAnimationFrame(loop);
  }

  // Frame index from continuous time
  getFrameIndex(time: number): number {
    return Math.floor(time / this.frameIntervalMs) % this.totalFrames;
  }

  // Frame-accurate stepping
  advanceFrame(direction: 1 | -1 = 1): void {
    const currentFrameIndex = this.getFrameIndex(this._playbackTime);
    const nextFrameIndex = (currentFrameIndex + direction + this.totalFrames) % this.totalFrames;
    const nextTime = nextFrameIndex * this.frameIntervalMs;
    this.seek(nextTime);
  }

  // Loop boundary handling with modulo wrap
  private handleLooping(): void {
    if (this._isLooping) {
      if (this._playbackTime >= this.duration) {
        this._playbackTime = this._playbackTime % this.duration;
      } else if (this._playbackTime < 0) {
        this._playbackTime = this.duration + (this._playbackTime % this.duration);
      }
    }
  }
}
```

### Grid Store

Zustand store with spatial indexing for cell management.

```typescript
interface CellUIState {
  id: string;
  contentType: 'empty' | 'video' | 'mesh' | 'comment';
  videoId?: string;
  modelId?: string;
  isSynced: boolean;
  nametag?: string;
  videoMode?: 'original' | 'overlay' | '3d';
  linkedCellId?: string;
}

interface CellSpatialIndex {
  id: string;
  row: number;
  col: number;
  contentType: 'empty' | 'video' | 'comment';
  videoId?: string;
}
```

### Three.js Mesh Rendering

PKL data → Float32Array → BufferGeometry with efficient per-frame updates.

```typescript
// Load mesh data from API (parsed from PHALP pickle)
const frames: MeshFrameData[] = data.frames.map((frame: any) => ({
  frameNumber: frame.frameIndex,
  // Flatten nested vertex arrays: [[x,y,z], ...] → Float32Array
  vertices: new Float32Array(
    Array.isArray(frame.meshData.vertices[0])
      ? frame.meshData.vertices.flat()  // [[x,y,z],...] → [x,y,z,...]
      : frame.meshData.vertices
  ),
  // Triangle indices for mesh topology (static across frames)
  faces: new Uint32Array(
    Array.isArray(frame.meshData.faces[0])
      ? frame.meshData.faces.flat()
      : frame.meshData.faces
  ),
}));

// Efficient geometry updates without recreation
const updateMeshGeometry = (frame: MeshFrameData) => {
  if (!meshRef.current) {
    // First frame: create geometry once
    const geometry = new THREE.BufferGeometry();
    const posAttr = new THREE.BufferAttribute(frame.vertices, 3);
    geometry.setAttribute('position', posAttr);
    geometry.setIndex(new THREE.BufferAttribute(frame.faces, 1));
    geometry.computeVertexNormals();
    
    const mesh = new THREE.Mesh(geometry, material);
    mesh.rotation.set(Math.PI/2, Math.PI/2, Math.PI/2); // SMPL → Three.js coords
    sceneRef.current.add(mesh);
    meshRef.current = mesh;
    return;
  }

  // Subsequent frames: update position attribute only (no GC)
  const posAttr = meshRef.current.geometry.getAttribute('position');
  posAttr.copyArray(frame.vertices);  // Copy 6890 * 3 floats
  posAttr.needsUpdate = true;         // Flag for GPU upload
};
```

## Backend Implementation


### SMPL Mesh Computation (PKL → Vertices)

Transforms PHALP pickle output into renderable mesh data using SMPL body model.

```python
def compute_smpl_vertices(smpl_params):
    """
    Transform SMPL parameters (pose + shape) into 6890 mesh vertices.
    
    SMPL model: V = W(T_P(β, θ), J(β), θ, W)
    - β (betas): 10 shape coefficients controlling body proportions
    - θ (body_pose): 23 joint rotations as 3x3 rotation matrices
    - global_orient: Root rotation (pelvis)
    """
    global_orient = torch.tensor(smpl_params['global_orient'].reshape(1, 3, 3))
    body_pose = torch.tensor(smpl_params['body_pose'].reshape(1, 23, 3, 3))
    betas = torch.tensor(smpl_params['betas'].reshape(1, 10))
    
    with torch.no_grad():
        output = smpl_model(
            global_orient=global_orient,
            body_pose=body_pose,
            betas=betas
        )
    
    return output.vertices[0].cpu().numpy()  # (6890, 3) mesh vertices
```

### 3D → 2D Projection (Camera Transform)

Converts crop-space camera to full-image coordinates for overlay rendering.

```python
def cam_crop_to_full(cam_bbox, box_center, box_size, img_size, focal_length=5000.):
    """
    Convert HMR2 camera params from crop space to full image space.
    
    cam_bbox: [s, tx, ty] - scale and translation in crop
    Returns: [tx, ty, tz] - camera translation in full image
    """
    img_w, img_h = img_size[:, 0], img_size[:, 1]
    cx, cy, b = box_center[:, 0], box_center[:, 1], box_size
    w_2, h_2 = img_w / 2., img_h / 2.
    
    bs = b * cam_bbox[:, 0] + 1e-9  # Scaled box size
    tz = 2 * focal_length / bs      # Depth from scale
    tx = (2 * (cx - w_2) / bs) + cam_bbox[:, 1]
    ty = (2 * (cy - h_2) / bs) + cam_bbox[:, 2]
    
    return torch.stack([tx, ty, tz], dim=-1)

def project_vertices_to_2d(vertices, cam_t_full, focal_length, img_size):
    """Perspective projection: 3D vertices → 2D pixel coordinates"""
    fx = fy = focal_length
    cx, cy = img_size[0] / 2.0, img_size[1] / 2.0
    tx, ty, tz = cam_t_full
    
    # Camera-space transform with Y/Z flip for OpenGL convention
    x_cam = vertices[:, 0] - tx
    y_cam = -(vertices[:, 1] - ty)
    z_cam = tz - vertices[:, 2]
    
    # Perspective divide
    x_2d = fx * x_cam / z_cam + cx
    y_2d = fy * y_cam / z_cam + cy
    
    return np.stack([x_2d, y_2d], axis=1)
```

### Python Pose Service Integration

Flask service wrapping 4D-Humans HMR2 for real-time 3D pose detection.

```typescript
export async function detectPoseHybrid(
  imageBase64: string,
  frameNumber: number
): Promise<HybridPoseFrame> {
  const response = await axios.post(`${POSE_SERVICE_URL}/pose/hybrid`, {
    image_base64: imageBase64,
    frame_number: frameNumber
  }, { timeout: 120000 }); // 2 min for first run (downloads ~500MB model)

  return {
    frameNumber: response.data.frame_number,
    keypoints: response.data.keypoints,           // 24 SMPL joints
    has3d: response.data.has_3d,
    joints3dRaw: response.data.joints_3d_raw,     // Raw 3D positions
    jointAngles3d: response.data.joint_angles_3d, // Knee, hip, spine angles
    mesh_vertices_data: response.data.mesh_vertices_data, // 6890 SMPL vertices
    mesh_faces_data: response.data.mesh_faces_data
  };
}
```

### Video Streaming with Range Requests

Supports seeking in large video files.

```typescript
app.get('/videos/:videoId', (req, res) => {
  const stats = fs.statSync(videoPath);
  const range = req.headers.range;

  if (range) {
    const [start, end] = range.replace(/bytes=/, '').split('-').map(Number);
    res.status(206);
    res.setHeader('Content-Range', `bytes ${start}-${end}/${stats.size}`);
    fs.createReadStream(videoPath, { start, end }).pipe(res);
  } else {
    res.setHeader('Accept-Ranges', 'bytes');
    fs.createReadStream(videoPath).pipe(res);
  }
});

---

## AWS Infrastructure 

- **ECS Cluster** on Fargate (serverless containers)
- **ECR** for Docker image registry
- **Application Load Balancer** with health checks
- **VPC** with public/private subnets across 2 AZs
- **Secrets Manager** for MongoDB credentials
- **CloudWatch Logs** with 7-day retention
- **S3 + CloudFront** for video/frame storage and CDN

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | React, Vite, TypeScript, Three.js, Zustand, Tailwind |
| Backend | Node.js, Express, MongoDB, Redis |
| ML/AI | 4D-Humans, ViTDet, PHALP, Gemini 1.5 Flash |
| Infrastructure | Docker, Terraform, AWS (ECS, ECR, ALB, S3, CloudFront) |

---
```
```
## Key Components

frontend/
├── engine/
│   └── PlaybackEngine.ts      # RAF loop, timing, events
├── stores/
│   └── gridStore.ts           # Cell state, spatial index
├── components/
│   ├── GridLayout.tsx         # Grid container
│   ├── GridCell.tsx           # Individual cell
│   ├── MeshViewer.tsx         # Three.js 3D viewer
│   ├── MeshScrubber.tsx       # Independent mesh controls
│   ├── CellScrubber.tsx       # Video scrubber
│   ├── VideoDisplay.tsx       # Video element wrapper
│   └── comments/              # Comment system
├── hooks/
│   ├── useMeshSampler.ts      # Mesh frame updates
│   ├── useVideoMeshSync.ts    # Video-mesh sync
│   └── useFrameCache.ts       # LRU frame cache
└── services/
    ├── meshDataService.ts     # API client with polling
    └── globalCameraManager.ts # Camera presets

backend/
├── src/
│   ├── server.ts                 # Express server, video streaming, CORS
│   ├── services/
│   │   ├── videoProcessingService.ts # track.py subprocess spawning
│   │   ├── pythonPoseService.ts      # 4D-Humans Flask client
│   │   ├── meshDataService.ts        # MongoDB mesh storage
│   │   ├── pickleParserService.ts    # PHALP output parsing
│   │   └── frameStorageService.ts    # Frame persistence
│   └── api/
│       ├── pose-video.ts             # Video upload + background jobs
│       ├── mesh-data.ts              # Mesh data retrieval
│       ├── videos.ts                 # Video CRUD
│       └── comments.ts               # Timeline comments
├── pose-service/
│   ├── app.py                        # Flask API (4D-Humans wrapper)
│   └── video_processor.py            # HMR2 inference
└── docker/                           # Container configs
```

---

## Impact

- Frame-accurate video comparison for coaching analysis
- 60fps synchronized playback across multiple videos
- Real-time 3D pose visualization from any angle
- Timestamped comments for coaching feedback
- Cost-optimized: ~$5/month for 1000 users
- Computer vision features are gated for full students only

---

## Future Work & Ideas

### Intelligent Video & Pose Caching

- Hash video file bytes to create a deterministic cache key
- Background job queue for GPU-heavy pose processing with status tracking

### Canonical Trick Models

- Build a library of "ideal" trick executions (e.g. backside 720) as pose sequences over time
- Compare rider pose data against canonical trick sequences frame-by-frame
- Measure joint angles, rotation timing, and body alignment relative to ideal form
- Use this as the foundation for all coaching feedback

### Visual Coaching Aids

- Ghost overlays of ideal pose sequences layered over rider video
- Joint angle arcs, rotation axes, and body orientation indicators
- Per-frame metrics highlighting where form deviates most
- Loopable comparison views for focused drill-down on takeoff, rotation, and landing phases

### Coach-Driven Feedback Flow

- Coaches generate shareable links that preload aligned, side-by-side comparisons
- No setup required for the student — link opens directly into scrub-ready analysis
- Query parameters define alignment, loop range, playback speed, and camera view
- Designed for quick reviews on the chairlift or between runs

### Toward an AI Snowboard Coach

- Break tricks into understandable phases (approach, takeoff, rotation, landing)
- Evaluate pose deviations per phase instead of treating tricks as a single motion
- MCP-driven coaching logic to map detected issues to fundamental snowboard concepts
- Automatically recommend drills and reference videos based on detected form issues

---

## License

MIT
