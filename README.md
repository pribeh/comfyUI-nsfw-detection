# comfyUI-nsfw-detection

A ComfyUI custom node for detecting and filtering NSFW content using NudeNet. This fork adds JSON metadata export capabilities for downstream processing.

## Features

- **NudenetDetector**: Original node that detects NSFW content and pixelates the image
- **NudenetDetectorMeta**: New node that provides both processed image AND detection metadata as JSON

## Installation

1. Clone this repository into your ComfyUI custom nodes directory:
```bash
cd ComfyUI/custom_nodes/
git clone https://github.com/pribeh/comfyUI-nsfw-detection.git
```

2. Install dependencies:
```bash
pip install -r requirements.txt
```

## Nodes

### NudenetDetector

The original detector with optional JSON export.

**Inputs:**
- `image` (IMAGE, required): Input image to analyze
- `export_json` (BOOLEAN, optional): Whether to export detection metadata as JSON (default: False)

**Outputs:**
- `image` (IMAGE): Processed image with NSFW content pixelated
- `detections_json` (STRING): JSON string containing detection metadata (empty array if export_json is False)

### NudenetDetectorMeta

Enhanced detector that always exports JSON metadata.

**Inputs:**
- `image` (IMAGE, required): Input image to analyze

**Outputs:**
- `image` (IMAGE): Processed image with NSFW content pixelated
- `detections_json` (STRING): JSON string containing ALL detection metadata (not just filtered NSFW content)

## JSON Output Format

The detection metadata is exported as a JSON array with the following structure:

```json
[
  [
    {
      "label": "FEMALE_BREAST_EXPOSED",
      "score": 0.98,
      "box": [123, 45, 260, 320]
    },
    {
      "label": "FEMALE_GENITALIA_EXPOSED",
      "score": 0.87,
      "box": [150, 180, 200, 250]
    }
  ]
]
```

Each detection includes:
- `label`: The classification label (e.g., "FEMALE_BREAST_EXPOSED", "BUTTOCKS_EXPOSED")
- `score`: Confidence score (0-1)
- `box`: Bounding box as `[x, y, width, height]`

## Available Detection Labels

The model can detect 18 different body part classifications:

- `FEMALE_GENITALIA_COVERED`
- `FACE_FEMALE`
- `BUTTOCKS_EXPOSED`
- `FEMALE_BREAST_EXPOSED`
- `FEMALE_GENITALIA_EXPOSED`
- `MALE_BREAST_EXPOSED`
- `ANUS_EXPOSED`
- `FEET_EXPOSED`
- `BELLY_COVERED`
- `FEET_COVERED`
- `ARMPITS_COVERED`
- `ARMPITS_EXPOSED`
- `FACE_MALE`
- `BELLY_EXPOSED`
- `MALE_GENITALIA_EXPOSED`
- `ANUS_COVERED`
- `FEMALE_BREAST_COVERED`
- `BUTTOCKS_COVERED`

## Automatic Pixelation Thresholds

The following classes trigger automatic pixelation when their confidence score exceeds the threshold:

- `BUTTOCKS_EXPOSED`: 0.7
- `FEMALE_BREAST_EXPOSED`: 0.45
- `FEMALE_GENITALIA_EXPOSED`: 0.45
- `ANUS_EXPOSED`: 0.45
- `MALE_GENITALIA_EXPOSED`: 0.45

## Usage in ComfyUI Workflows

### Basic Usage (Original Node)

Simply connect an image to the `NudenetDetector` node to get a pixelated output if NSFW content is detected.

### With JSON Export

1. Use `NudenetDetectorMeta` node and connect your image
2. Connect the `detections_json` output to a `Save Text` or similar node
3. Configure the text save node to write to the same output directory with a `.nsfw.json` extension
4. Example filename pattern: `ComfyUI_00001_.nsfw.json` (matching image `ComfyUI_00001_.png`)

### Integration with Worker Services

For automated workflows (e.g., with RunPod or similar services):

1. Configure your ComfyUI workflow to use `NudenetDetectorMeta`
2. Save the JSON output with the same base filename as the image
3. Your worker script can then load and attach the metadata:

```python
import json

def load_nsfw_metadata(filename, subfolder):
    """Load NSFW detection metadata for an image"""
    base = os.path.splitext(filename)[0]
    nsfw_path = os.path.join(output_dir, subfolder, f"{base}.nsfw.json")
    
    if os.path.exists(nsfw_path):
        with open(nsfw_path, 'r') as f:
            return json.load(f)
    return None

# In your image processing loop:
nsfw_data = load_nsfw_metadata(filename, subfolder)
if nsfw_data:
    image_record["nsfw"] = nsfw_data
```

This allows your server to:
- Filter content based on detection scores
- Apply content warnings
- Implement age-gating
- Track and moderate user-generated content

## Original Repository

This is a fork of [katalist-ai/comfyUI-nsfw-detection](https://github.com/katalist-ai/comfyUI-nsfw-detection) with added JSON export functionality.

## License

Same as the original repository.
