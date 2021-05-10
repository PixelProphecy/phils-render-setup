# phils-render-setup
A Game Maker Studio 2 Normal Lighting Setup
![Phil's Render Setup](/github-assets/prs-title-image.png?raw=true "Phil's Render Setup")

# Phil's GML Renderer
A simple 2D lighting & rendering system for Game Maker with support for Diffuse Shading, Normal Maps, Phong Shading, and Lens Flares and up to 8 dynamic light sources.

Because I was so fed up with a normal-mapping "tutorial" and demo project that did what they could to confuse newcomers, I decided to start anew with my own demo and share it with the world. It's not the fastest system out there, but perhaps a good starting point for your own experiments. Feel free to explore and tinker; I did my best to explain how things are working in this document.
 
Pixel Prophecy, May 2021
Version 1.1.0

Hi there!

If you're reading this, welcome! This document is mostly intended for myself to know how things are supposed to work and what the gotchas are. If there's a gotcha or other important knowledge, I prefix it with **[!]** . After the "Overall" explanation, there's a F.A.Q. to hopefully answer questions about how you, yes *you*, can use it for your own projects. Finally there's a "Caveats" section that you know what you get yourself into.
# The Demo Project
Attached is a GameMaker Studio 2 demo project that demonstrates everything in action. Here's what you can do:
- **Left Click**: Place current light and create a new one
- **Right Click**: Remove last light
- **WASD**: Move view
- **Mouse Wheel**: Change the z-depth of a light source
- **Ctrl + Mouse Wheel**: Flip though the individual render passes  
# How it works
In general, each "renderable" object has three sprites, one each for diffuse, normals, and specularity. The "renderer" now draws the sprites to four surfaces (diffuse, normals, specularity, and depth) and a big shader combines them all to produce the resulting image. 

I tried to keep the code together as much as possible that it's easy for others to expand upon and understand what's happening at what stage.

Onwards!

# Overall

Right. For the basic functionality you need:
1. The **renderer** object (*ctrl_renderer*)
2. The **phong shader** (*sh_normal_phong*)
3. The **depth shader** (*sh_depth*)
4. The renderable **class** object (*CLASS_renderable*)
5. The **light** object (*obj_r_light*)
6. Rendering **functions** script (*scr_rendering*)

Additionally there are objects such as
- ***ctrl_camera*** that allows you to move the view around with WASD
- ***ctrl_init*** , the initialization object. All it does is create an instance of ctrl_renderer and sets a global variable
- ***obj_fx_flare*** , the lens flare object that draws a flare. 
- ***obj_pointer***, for handing the mouse to create light sources and have them follow the mouse pointer

Let's address them one by one and how they interact.

## 1. The Renderer
This is where most of the settings regarding rendering are kept and initialized. You want this object to exist, when you want to use the features of the shader.

Overall, the renderer keeps a list of dynamic lights, arrays that get filled up for the use in the shader with light properties (more on that in the *Draw* event), handles to the surfaces of the individual passes and handles to the shader.

### Create Event
 - Here we set a few global variables, all prefixed with "R_" to denote that they belong to rendering. These global variables only pertain management of the lights so that the rendering functions script can easily access them.
 - **[!]** global.R_max_dynamic_lights is independent from the max lights in the shader! So if you change this value, you also need to change the line
`#define LN 8 // Max number of lights`
in the *sh_normal_phong* fragment shader!
 - The ambiance color and intensity also gets defined here. The array stores the RGB values of the light in the shader-friendly 0.0-1.0 range and gets set each step.
 - The falloff defines the general falloff behavior of all lights. The greater the values are apart, the stronger it will be. E.g. [.1, .5, 1] feels pretty linear, a stronger one could be [.4, 2, 20].
 - Also, the camera's position and view get set here and updated every step, just to be safe.
 - The enum *RENDER_PASS* is currently only used for debugging.
 - All handles to the shader uniforms are prefixed with "shader_u_"
 - Currently, *shader_u_resolution* is unused.
 
### Step Event
 - Each step, the ambience color gets stored in the *ambience[]* array 
 - The current camera view x, y, width and height get stored.
 - Debugging: The mouse wheel changes the current pass to be displayed in the shader, you can remove this.

### Draw Begin Event
- First the renderer gather all objects of *CLASS_renderable* and sorts them by their z-Depth, so that they get drawn in the right order, no matter on what layer they might be on. For further explanation of the z-Depth, refer to the section about *CLASS_renderable*
- The renderer makes sure that for each pass a surface exists (diffuse, normal, specularity) and instructs each *CLASS_renderable* child to draw its according sprite onto the correct surface.
- This means that drawing happens three times here already, a fourth time draw event.
- **[!]** Each sprite gets drawn using the internal variables `image_index, x, y, image_xscale, imagey_scale`, and `image angle`, so always use these in deviates from the *CLASS_renderable* instead of rolling your own.
- **[!]** The normals sprite ignores the `image_angle` and is never rotated because that would change the normals, of course.

**[!]** The specular coefficient (= how sharp the Phong highlights will be) are ***stored as alpha*** in the sprites but that causes issues with blending. The best result is if the source alpha completely replaces the destination alpha (`gpu_set_blendmode_ext(bm_src_alpha, bm_zero)`) but that makes it impossible to have transparent parts in a sprite, they will always be in rectangles. A solution may be to write a shader that draws the phong coefficient to the alpha of the specular surface but multiplies it with the alpha in the diffuse channel first...

### Draw Event
Here the important parts happen:
- First, all lights are gathered and their current state and properties are written to the global arrays:
- - *R_lights* stores x, y, and z of each light, so if you have 3 lights active, the array will have 9 entries.
- - The same happens with *R_lights_color*, for each light it stores the RGB values but converted to the shader-friendly 0.0-1.0 range.
- - *R_lights_RIS* stored 3 additional values per light in the following order: range, intensity, specularity.
- After the "gather lights" block, all global light arrays should be current and the *lights_count* variable should hold the current number of lights.
- Next there's a switch of the current pass. That's only for debugging, so it can be commented out.
- Then we set the shader to *shader_normal_phong*,
- pass it the pointers to all the passes surfaces (diffuse, normal, spec),
- hand it the falloff array, the ambience color and intensity, the number of currently active dynamic lights, and the arrays with the light properties. 
- Also, the current pass. Again, used for debugging, remove.
- Finally, the diffuse surface is drawn with the shader but it can be any surface.

### Draw GUI
There's just debugging stuff, can be removed entirely.

## 2. The Phong Shader
The shader is one of those monsters you don't want in your game because it tries to do everything. The more lights there are, the more costly it gets. But since it's the only one shader, I think that's acceptable.

### Vertex Shader
In the **Vertex** part it's mostly standard stuff. Though, it's important to also get the *v_vPosition* as *vec2* and pass it on to the fragment shader.

**[!]** Also note that color is written without the "u" in both vertex and fragment parts.
 
### Fragment Shader
The fragment shader has a lot going for itself. Basically it takes all the passes (diffuse, normal, spec, and depth) as sampler2D and all the lights and their properties as float arrays.

- First, the view direction is always the same, we're looking straight onto the scene. This will be used in the Phong formula further down.
- We sample all the maps to vec4, but since the depth doesn't need to be, we take the red component as float.
- Then we get the ambient color and multiply it with the diffuse.
- Next we convert the normal map.
- **[!]** Some programs invert the green channel on the normals, so (un)comment `//N.g *= -1.0;` for that, but this is global, of course.
- Next up is a for-loop for each light:
- We get the property of each one from the arrays that were passed, get the distance and length.
- **[!]** The length is normalized but it should not be for the falloff to work correctly. But if it's not normalized, it doesn't work, so normalized it stays.
- The *LinearAttenuation* is there to force a linear light falloff on top of the attenuation, otherwise lights are just to far-reaching.
- The specular color of the material gets sampled from the spec map and multiplied with the light color.
- **[!]** Light specularity is only one float, so it's not possible for a light to have its specularity in a color different to its own. Shouldn't be necessary anyway.
- The `//if (dot(N, L) > 0.0) // light source on the right side?` line is commented out because it seems like it's not needed and if-conditions are very costly in a shader.
- The *specularReflection* is now the Phong formula.
- You'll notice that the pow() can't be below "0.001" since if it's 0.0, ugly black artifacts happen when multiple lights overlap.
- Under the comment `// the calculation which brings it all together` are three versions of calculating the Intensity, two of which use the *LinearAttenuation*. Pick the one that works best for you, normally it should be `vec3 Intensity = Diffuse * Attenuation * (LinearAttenuation + specularReflection);` The lowest one has the biggest performance impact, the top-most one the least.
- Finally, resulting pixels get added to the existing diffuse map color.
- The big if-block is for debugging, so that you can use Ctrl+Mouse Wheel to switch between the passes. This should be taken out in the "final" version because if-statements are costly, remember?
- At the end we output the final color completely opaque, so no need for alpha.

## 3. The Depth Shader
This shader is only used when *ctrl_renderer* draws the depth pass. All it does is to draw a sprite, filled with the passed value for zDepth. It also turns the alpha into a 1-bit value because anti-aliasing would produce incorrect depth information in the depth pass. 

## 4. The Renderable Class
This is a class that all things you want to be affected by light should derive from, i.e. be a child of this class.

### Create Event
It has only this event in which variables for the individual sprites for diffuse, normal, and spec, are referenced.

Ideally, follow the naming convention of prefixing it with "obj_r_" so the r stands for "renderable".

#### Z-Depth
This is important: With my *ctrl_renderer*, the internal *depth* variable gets pretty much ignored, you need to set each *z* property manually. 0.0 means that it's furthest from the camera, 1000.0 means it's closest. Avoid values below 0 and above 1000 for this z-value, as it will get divided by 1000 when creating the depth map and passed to the depth shader. Each step the renderer sorts the renderables by depth.

## 5. The Light Object
The object is called "obj_r_light", the "r" implying that it's a "rendering" object.

### Create Event
We have the usual suspects. 

**[!]** If you want to create a light, don't use *instance_create_depth()*, instead do it with a call to *RENDER_add_light()*, which will return the instance id of the newly created light. 

If you want to set a light's properties, be sure to set the variables ending with "_init", e.g. `light.falloff_init = 800;`

It's also possible to constrain a light to another instance and have it automatically follow its position. For that you create the light, set its *owner* to the *id* of the object to follow and set the *is_owned* flag to true. That way the light will automatically follow its owner and destroy itself when the owner doesn't exists.
Example:
```
  var light = RENDER_add_light();
  light.owner = obj_mouse;
  light.is_owned = true;
```
**[!]** Don't forget to set the *z* property, the depth of the light. This is important so that shader can place it correctly. Same goes for the *z*-value in *CLASS_renderable* and its children.
 
### Step Event
Here, the light just checks whether it has an owner and if so, to follow its translation. When it gets created, the *do_init* block runs once and there it creates a lens flare for itself.
 
## 6. Rendering Functions Script

In this script file are all functions related to rendering in one place. To make this very obvious, all functions are prefixed with "RENDER_". I tried to keep it as small as possible, so a lot of the important stuff happens in *ctrl_renderer*. 

Currently there are two functions and they deal with adding and removing light sources.

**RENDER_add_light()** creates a new light and registers it in the global list of currently active dynamic lights, then returns the instance id. Currently, the handling of more lights than the maximum could be improved!

**RENDER_remove_light(instance id)** tries to remove the passed light from the list and if it doesn't find it, it removes the last light and destroys the instance.

Not perfect (for now) but gets the job done so that there aren't too many lights active. A good maximum should be around 8.

# F.A.Q.
## How can I add my own object affected by lighting?

1. Create a new object and make it a child of *CLASS_renderable*.
2. Ideally, follow the naming convention of prefixing it with "obj_r_" so the r stands for "renderable".
3. Inherit the *Create* event and set the following variables with the correct sprites. If you don't have a sprite, keep *noone*: 
```
sprite_diffuse = noone;
sprite_normal  = noone; 
sprite_spec    = noone
```
4. Either explicitly or programmatically set a depth value for z, e.g.
`z = 100`. Remember, this *z* is different from the built-in *depth* and ranges from 0.0 (furthest away) to 1000.0 (closest to the camera)

## How can I create a light?
To create a light and have it properly handled by the renderer, use the **RENDER_add_light()** function from *scr_rendering*. It returns the instance id of the light. With that you can set its properties. For example:
```
var l = RENDER_add_light();
l.color_init     = c_aqua;
l.intensity_init = 1;
l.falloff_init   = 800;
l.x      = x;
l.y      = y;
l.z_init = 100;
```
If you want to remove a light, call RENDER_remove_light(instance id of light).

## How can I remove a light?
Pass the instance *id* of the light to remove to **RENDER_remove_light()**, e.g. `RENDER_remove_light(current_light);`. If you don't specify a light, use *noone* which will remove the last entry in the global list of dynamic lights.

## How can I make textures?
Diffuse and normal should be straight forward. As for the spec map, it's important to know that in the RGB the spec color gets stored, you can tweak the intensity by reducing the brightness.

The alpha channel stores the Phong coefficient, meaning how large the specularity will be. 0 is super wide, 255 is a very sharp and narrow highlight. As outlined in the "Draw Begin" Section of *ctrl_renderer*, above, try to keep it in invisible parts of a sprite at 0.

**[!]** In Tools > Texture Groups you need to un-tick "Automatically Crop" in all texture groups with specularity sprites because otherwise GameMaker clips away all regions with alpha 0, but for spec sprites we need this information.

# Caveats 
That's all nice and dandy, you might think to yourself, but where's the catch? What am I getting myself into? 
- It's not super performant, I have to admit. The more dynamic lights you have, the more your GPU will begin to choke. It's all down to the poorly optimized shader and that it samples four surfaces per pixel. Add to that the overdraw of all the lens flares and you get beautiful albeit slow graphics.
- If you want to have static lights, there's no way of having them in this implementation. Maybe it would be possible to render them onto a surface once and keep that surface around but I haven't given it much thought.
- You *need* to handle depth for all renderables yourself, meaning to get used to working with the *z* property of *obj_r_light* and *obj_renderable* and its children. always keep it between 0 and 1000.
- Creating specular maps might be tedious. As outlined above, the Phong-coefficient, the “shininess” of a material is stored in the sprite's alpha channel, 0 meaning completely matte, 255 is super glossy. It's a bit annoying to create sprites that way and even more annoying to view them, as most of the time they will look wispy and almost transparent.
- Speaking of specular maps: Drawing them on the spec pass is buggy precisely because of their alpha. I tried using the blending mode `gpu_set_blendmode_ext(bm_src_alpha, bm_inv_src_alpha)`but i don't think that it's correct. What you want is to use the alpha map from the diffuse sprite to draw the alpha of the specularity, if that makes sense, much like the shader *sh_depth*. That could be your homework! ;)

Thank you for reading this lengthy document. I hope I could provide you with some information and explain things in a way that you know how the system works and make your own changes. 

Please consider buying me a coffee over at pixelprophecy.com/donate :)
