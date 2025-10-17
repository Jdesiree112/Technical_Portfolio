# Memory Optimization
## Overview
Over the course of the implementation of an agentic workflow, as detailed in ["Prompt Library Architecture Redesign"](https://github.com/Jdesiree112/Technical_Portfolio/blob/main/CaseStudy_Mimir/Issue_ProblemSolvingCaseStudy/Issue-Solutions_Entry009.md), the memory limit for Mimir's HuggingFace space was exceeded. Provided that this project is just a demo meant to allow for the demonstration of advanced prompt engineering implementation and library curation, it is not practical or reasonable to invest additional funds for maintaining this space as is. Rather, I have instead opted to optimize the memory usage. 

## Actions

### Removed Item
#### Phi3 Base Model Fallback
The base model used by Mimir's fine-tuned response model was originally compiled during build alongside the fine-tuned model and Mistral reasoning models, cached for later use. This had originally been implemented due to issues with the fine-tuned model loading properly while under development. The loading issues have been resolved, rendering the base model unused but still occupying memory space and adding extra complexity through fallback logic. For these reasons, the base model and all instances of reference have been removed from Mimir.

### GPU Memory Logging and Sequential Model Loading
Logging has been added to `agents.py`, combining a log_memory() + nvidia-smi approach, to track GPU usage per step. Additionally, model loading has been set to proceed sequentially, negating potential issues with GPU VRAM being maxed out by too many calls at once.
Corresponds to [commitment 7417fd2e2d5b5a9297c7dacd250fdd4715dcec98](https://huggingface.co/spaces/jdesiree/Mimir/commit/7417fd2e2d5b5a9297c7dacd250fdd4715dcec98)

### Monitoring
A new analytics tab was added to the trackio page of Mimir's application, specifically to view the cache size of the application. 

```
def show_cache_info():
    try:
        from pathlib import Path
        from huggingface_hub import scan_cache_dir
        
        cache_info = scan_cache_dir(cache_dir="/tmp/huggingface")
        
        info_text = f"""
**HuggingFace Cache Status:**

**Total Size:** {cache_info.size_on_disk / (1024**3):.2f} GB
**Number of Repos:** {len(cache_info.repos)}

**Cached Models:**
"""
        
        for repo in cache_info.repos:
            size_gb = repo.size_on_disk / (1024**3)
            info_text += f"""
- **{repo.repo_id}**
  - Size: {size_gb:.2f} GB
  - Type: {repo.repo_type}
  - Revisions: {len(repo.revisions)}
"""
        
        return info_text
        
    except Exception as e:
        return f"Error inspecting cache: {str(e)}"
```
