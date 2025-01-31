# stable-diffusion-webui-prompt-travel

    Extension script for AUTOMATIC1111/stable-diffusion-webui to travel between prompts in latent space.

----

This is the more human-sensible version of [stable-diffusion-webui-prompt-erosion](https://github.com/Kahsolt/stable-diffusion-webui-prompt-erosion), 
now we do not modify on text char level, but do linear interpolating on the hidden embedded vectors. 😀  

⚠ 我们成立了插件反馈 QQ 群: 616795645 (赤狐屿)，欢迎出建议、意见、报告bug等 (w  
⚠ We have a QQ chat group now: 616795645, any suggeustion, discussion and bug reports are highly wellllcome !!  

ℹ 实话不说，我想有可能通过这个来做ppt童话绘本<del>甚至本子</del>……  
ℹ 聪明的用法：先手工盲搜两张好看的图 (只有prompt差异)，然后再尝试在其间 travel :lolipop:  


### Change Log

- 2022/11/14: walk by substituting word embedding ('replace' mode)
- 2022/11/13: walk by optimizing condition ('grad' mode)
- 2022/11/10: interpolate linearly  on condition/uncondition ('linear' mode)


### How it works?

- generate image one by one (batch configs are ignored)
- gradually change the digested inputs between prompts
  - freeze all other settings (steps, sampler, cfg factor, rand seed, etc.)
  - force `subseed == seed, subseed_strength = 0.0`
- gather to be a video!

**DDIM**:

![DDIM](img/ddim.gif)

**Eular a**:

![eular_a](img/eular_a.gif)


### Options

- prompt: (list of strings)
- negative prompt: (list of strings)
  - we call each line of prompt a stage
  - if len(postive_prompts) != len(negative_prompts), the shorter one's last item will be repeated to match the longer one
- mode: (categorical)
  - linear: interpolate linearly on condition/uncondition in latent space
  - replace: walk by gradually substituting word embededings 
  - grad: walk by optimizing certain loss (see [Experimental](#experimental))
  - NOTE: `walk` methods might not reach target stages in specified steps, manually tune `grad_alpha` or increase `steps` in that case accroding to log losses...
- steps: (int, list of int)
  - number of images to interpolate between two successive stages<del>, set `-1` to allow wanderding util converge for `walk` methods (not yet implemented)</del>
  - if int, constant number of travel steps
  - if list of int, length should match `len(stages)-1`, separate by comma, e.g.: `12, 24, 36`
- replace_*
  - replace_order: (categorical)
    - `random`: substitute tokens randomly
    - `similiar`: substitute most similiar tokens first (L1 distance of token embeddings)
    - `different`: substitute most diffrent tokens first (L1 distance of token embeddings)
- grad_*
  - grad_alpha: (float), step size of a walk pace
  - grad_iter: (int), step count of walk paces
    - you can try trading `grad_alpha=0.01 grad_iter=1` for `grad_alpha=0.001 grad_iter=10`
    - might be more cautious (perhaps!), but definitely takes more time
  - grad_meth: (categorical), step function of a walk pace
    - `clip`: a triky balance between `sign` and `tanh`
    - `sign`: walk at a constant speed (often stucks into oscillation at the end)
    - `tanh`: significantly speed down when approching (it takes infinite time to exactly reach...)
  - grad_w_latent: (float), weight factor of `loss_latent`
  - grad_w_match: (float), weight factor of `loss_cond`
- fps: (float)
  - FPS of video, set 0 to disable saving
- debug: (bool)
  - whether show verbose debug info at console

⚠ this script will NOT support the schedule syntax (i.e.: `[prompt:prompt:number]`), because I don't know how to interpolate between different schedule plans :(  
⚠ max length diff for each prompts should NOT exceed `75` in token count, otherwise will only work on the first segment, cos' I also don't know how to interpolate between different-lengthed tensors 🤔  


### Installation

Easiest way to install it is to:
1. Go to the "Extensions" tab in the webui, switch to the "Install from URL" tab
2. Paste https://github.com/Kahsolt/stable-diffusion-webui-prompt-travel.git into "URL for extension's git repository" and click install
3. (Optional) You will need to restart the webui for dependencies to be installed or you won't be able to generate video files

Manual install:
1. Copy this repo folder to the 'extensions' folder of https://github.com/AUTOMATIC1111/stable-diffusion-webui
2. (Optional) Restart the webui


### Experimental

⚪ 'grad' mode

The `loss_latent` optimizes `mse_loss(current_generated_latent, target_latent)` 

  - if `grad_w_latent` is positive, minimizing
  - if `grad_w_latent` is negative, maximizing

The `loss_cond` optimizes `l1_loss(current_cond, next_stage_cond)`

  - if `grad_w_cond` is positive, walk towards the next stage (minimizing)
  - if `grad_w_cond` is negative, walk away from it (maximizing)

Grid search results: (`steps=100, grad_alpha=0.01, grad_iter=1, grad_meth='clip'`)

| w_cond\w_latent | -1 | 0 | 1 |
| :-: | :-: | :-: | :-: |
| -1 | 纹理丢失色块平滑、逆向胚胎发育，最后变成圆圈堆叠成的抽象小人 | 前几步变得精致，随后纹理丢失色块平滑，但保持作画结构，中途突然高斯模糊，旋即背景失去语义，最后变成斑点图，l_grad下降 | 走到三张别的图，画风基本一致，背景变朦胧，途中震荡，最后人物没了，变得几何重复 |
| 0 | 纹理丢失色块平滑、逆向胚胎发育，最后变成圆圈堆叠成的抽象小人，l_l1上升 | - | 走到两张别的图，画风基本一致，背景变朦胧，途中震荡，l_l1上升 |
| 1 | 纹理丢失色块平滑、逆向胚胎发育，最后变成圆形蒙版、光栅纹理 | **近似线性插值，叠加式过渡到目标，途中震荡，l_grad下降** | 走到两张别的图，画风基本一致，背景变朦胧，最后震荡 |

(*) 上表如无特殊说明，其各项 loss 变化都符合设置的优化方向  
(**) 我们似乎应当总是令 `w_latent > 0`，而 `w_cond` 的设置似乎很玄学，这里可能遭遇了对抗样本现象(神经网络的过度线性性)……  

ℹ NOTE: When 'prompt' has only single line, it will wander just **around** the init stage, dynamically balancing `loss_latent` and `loss_cond`; this allows you to discover neighbors of your given prompt 😀

----

by Armit
2022/11/10 
