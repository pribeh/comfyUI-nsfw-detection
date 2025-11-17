# Integration Guide: JSON Metadata Export

## Changes Made

This fork of comfyUI-nsfw-detection has been enhanced to export NudeNet detection data in JSON format for integration with yuser-server.

### 1. Modified Files

- **`nudenet.py`**: Enhanced with JSON export capabilities
- **`README.md`**: Comprehensive documentation of new features

### 2. New Features

#### A. Enhanced `NudenetDetector` Node
- Now returns both `IMAGE` and `STRING` (JSON) outputs
- Added optional `export_json` boolean parameter (default: False)
- Backward compatible - works as before when `export_json` is False

#### B. New `NudenetDetectorMeta` Node
- Always exports JSON metadata
- Returns ALL detections (not just filtered NSFW content)
- Ideal for comprehensive content analysis

### 3. JSON Output Format

```json
[
  [
    {
      "label": "FEMALE_BREAST_EXPOSED",
      "score": 0.98,
      "box": [123, 45, 260, 320]
    }
  ]
]
```

Each detection contains:
- `label`: Classification (e.g., "FEMALE_BREAST_EXPOSED")
- `score`: Confidence score (0-1)
- `box`: Bounding box as `[x, y, width, height]`

## Next Steps: Docker Integration

To use this fork in your worker-comfyui Docker build:

### Option 1: Update Dockerfile to Clone from Fork

Find the line in your Dockerfile that clones the original repo and replace it:

**Original:**
```dockerfile
RUN git clone https://github.com/katalist-ai/comfyUI-nsfw-detection.git
```

**Updated:**
```dockerfile
RUN git clone https://github.com/pribeh/comfyUI-nsfw-detection.git
```

### Option 2: If Using Git Submodules

Update the submodule URL:
```bash
cd worker-comfyui
git submodule set-url custom_nodes/comfyUI-nsfw-detection https://github.com/pribeh/comfyUI-nsfw-detection.git
git submodule update --remote
```

### Option 3: Manual Installation During Build

Add to your Dockerfile:
```dockerfile
WORKDIR /comfyui/custom_nodes
RUN rm -rf comfyUI-nsfw-detection
RUN git clone https://github.com/pribeh/comfyUI-nsfw-detection.git
```

## Worker Integration

### 1. Update ComfyUI Workflow

In your ComfyUI workflow JSON:
- Use `NudenetDetectorMeta` node
- Connect the `detections_json` output to a `Save Text` node
- Configure filename pattern: `{filename_prefix}.nsfw.json`

### 2. Worker Script Enhancement

Add to your worker handler (handler.py):

```python
def load_nsfw_metadata(filename, subfolder, output_dir):
    """Load NSFW detection metadata for an image"""
    base = os.path.splitext(filename)[0]
    nsfw_path = os.path.join(output_dir, subfolder, f"{base}.nsfw.json")
    
    if os.path.exists(nsfw_path):
        try:
            with open(nsfw_path, 'r') as f:
                data = json.load(f)
                # Flatten if it's an array of arrays (batch processing)
                if data and isinstance(data[0], list):
                    return data[0]
                return data
        except Exception as e:
            print(f"Error loading NSFW metadata: {e}")
    return None

# In your image processing loop:
for filename in output_files:
    # ... existing image processing ...
    
    # Add NSFW metadata
    nsfw_data = load_nsfw_metadata(filename, subfolder, output_dir)
    if nsfw_data:
        image_record["nsfw"] = nsfw_data
        image_record_simple["nsfw"] = nsfw_data
```

### 3. Server-Side Filtering (yuser-server)

Example content filtering based on metadata:

```javascript
// In your content processing endpoint
function analyzeNSFWContent(nsfwData) {
  if (!nsfwData || !Array.isArray(nsfwData)) {
    return { isNSFW: false, maturityRating: 'G' };
  }
  
  const highRiskLabels = [
    'FEMALE_BREAST_EXPOSED',
    'FEMALE_GENITALIA_EXPOSED',
    'MALE_GENITALIA_EXPOSED',
    'BUTTOCKS_EXPOSED',
    'ANUS_EXPOSED'
  ];
  
  let maxScore = 0;
  let hasHighRisk = false;
  
  for (const detection of nsfwData) {
    if (highRiskLabels.includes(detection.label)) {
      hasHighRisk = true;
      maxScore = Math.max(maxScore, detection.score);
    }
  }
  
  return {
    isNSFW: hasHighRisk && maxScore > 0.45,
    maturityRating: hasHighRisk && maxScore > 0.7 ? 'MATURE' : 
                    hasHighRisk && maxScore > 0.45 ? 'TEEN' : 'G',
    maxConfidence: maxScore
  };
}

// Apply filtering
const analysis = analyzeNSFWContent(imageData.nsfw);
if (analysis.isNSFW) {
  // Apply age gate, blur, warning, etc.
}
```

## Testing

1. **Build Docker Image:**
   ```bash
   cd worker-comfyui
   docker build -t worker-comfyui:test .
   ```

2. **Test Workflow:**
   - Send a test job with an image
   - Verify `.nsfw.json` file is created
   - Check webhook payload includes `nsfw` field

3. **Verify Integration:**
   - Check yuser-server receives NSFW data
   - Test content filtering logic
   - Validate maturity ratings are applied

## Troubleshooting

**Issue: JSON file not created**
- Check ComfyUI workflow has `Save Text` node connected
- Verify output directory permissions
- Check ComfyUI logs for errors

**Issue: Empty JSON array**
- Confirm `NudenetDetectorMeta` is used (not `NudenetDetector` with export_json=False)
- Check if image actually has detectable content

**Issue: Worker not finding JSON file**
- Verify filename pattern matches between save node and loader
- Check subfolder paths are correct
- Ensure timing - wait for all files to be written

## Available Detection Labels

The model can detect 18 body part classifications. The most relevant for content filtering:

**High Risk (Auto-pixelated):**
- BUTTOCKS_EXPOSED (threshold: 0.7)
- FEMALE_BREAST_EXPOSED (threshold: 0.45)
- FEMALE_GENITALIA_EXPOSED (threshold: 0.45)
- ANUS_EXPOSED (threshold: 0.45)
- MALE_GENITALIA_EXPOSED (threshold: 0.45)

**Lower Risk:**
- FEMALE_BREAST_COVERED
- FEMALE_GENITALIA_COVERED
- BUTTOCKS_COVERED
- ANUS_COVERED

**Neutral:**
- FACE_FEMALE, FACE_MALE
- BELLY_EXPOSED, BELLY_COVERED
- FEET_EXPOSED, FEET_COVERED
- ARMPITS_EXPOSED, ARMPITS_COVERED
- MALE_BREAST_EXPOSED

## Benefits

1. **Content Moderation**: Automatically flag and filter mature content
2. **Age Gating**: Apply appropriate restrictions based on detection confidence
3. **User Safety**: Protect users from unexpected mature content
4. **Analytics**: Track content types across your platform
5. **Compliance**: Meet platform guidelines and legal requirements
