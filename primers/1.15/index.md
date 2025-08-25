# 1.14 -> 1.15 Update Primer

## License
Your choice of CC0 or MIT

## Not Rendering
* Honestly not much. Ask in the usual channels.

## Rendering
### New Forge model loader
* Forge Blockstate jsons are now removed as having model logic being was always a massive hack
* Instead, the vanilla model json format has been expanded. You can now specify a "loader" field in the json and register a
custom loader to process the model in code.

### Batching
In the *vast majority* of cases, you should no longer use direct OpenGL calls via GlStateManager/GL11. A good rule of thumb is if you have
a `MatrixStack` and `IRenderTypeBuffer` available (either from an Event or from method parameters), you are in batching mode and shouldn't make OpenGL calls.

#### Motivation
Why this change? First of all, it's more performant as the number of OpenGL state changes is reduced. Secondly, it allows
for a more error-safe design using the render layers (see next section). Finally, it's preparation for future modernizations
that will eventually disallow all non-batched renders (and potentially update to higher opengl versions)

#### RenderType/Render Layers
The new render system centers around `RenderType` and `BufferBuilder`. The latter is the same as the one found in good old `Tessellator`.
The former, however, is new. It encodes the entire GL state (as much as Minecraft cares about "entire") that is desired,
e.g. which texture is bound, the alpha and depth test states, the blending states, etc., as well as the desired vertex format and draw mode.
`IRenderTypeBuffer`, then, should be treated as a `Map<RenderType, BufferBuilder>`. Each render layer has its own buffer builder.
Now, when blocks, entities, and tile entities are drawn, each renderer retrieves the buffer corresponding to the desired render type, draws to it with the
usual pos/tex/color/endVertex etc. However, it *doesn't* call the hypothetical equivalent of `tessellator.draw()`.
The game does that at the end of the frame, batching everything of one render type together. It renders all the geometry
for layer 1, then all of the geometry for layer 2, and so on. Thus, we switch GL states at most `<number of rendertype>` times
instead of jumping around as different entity renderers/tesrs feel like changing it.
One additional benefit you might also notice is that you no longer have to worry about setting up the GL state properly for every scenario -- no more forgetting to `enableStandardItemLighting` before you render an item, for example, because the layers that items
use already set up this GL state themselves.

#### Rendering Batched Stuff
For "normal" scenarios, you'll mostly be responsible for passing the `MatrixStack` and `IRenderTypeBuffer` around from the 
entity/TE renderers to other places, like the font/item/model renderers. The vanilla code will take care of picking the
correct RenderType and drawing to the right buffer.
For more special or custom scenarios, you'll have to pick a RenderType and draw to the `BufferBuilder` yourself.
To pick a RenderType, look in `RenderType`, `RenderTypeLookup`, and `Atlases` to see if one of vanilla's fits your needs.
The methods taking a RL and returning a `RenderType` allow you to pass in a texture that will be bound, with the rest of
the GL state held constant based on the method.
For example, `RenderType.getEntityTranslucent(<RL>)` acts the same as `Atlases.getEntityTranslucent()`, but uses the texture passed in instead of the main block atlas.
Sometimes, you'll need to create your own `RenderType`s, see how vanilla does it for an example. You'll have to specify
the GL state you want this layer to be drawn under, and that GL state will automagically be set up and torn down for you.

#### Calling batched code from unbatched environments
Some environments are still unbatched (most notably, guis). If you need to call batched code from such an environment,
you can produce an `IRenderTypeBuffer.Impl` by calling `IRenderTypeBuffer.immediate` and passing it a scratch BufferBuilder
to use. The one from `Tessellator.getInstance().getBuffer()` will usually do. After calling the batched code, finish with
`IRenderTypeBuffer.Impl.draw()`.

#### Porting
Given the above, a series of quick heuristics for porting renderering code:
1. Update the method signatures. If you see new integer parameters, they are most likely the lightmap texture coordinates and the 
overlay texture coordinates. The first controls light (obviously), the second controls overlays can usually be ignored (see TheGreyGhost's comment below).
2. If you don't have access to a `MatrixStack` and `IRenderTypeBuffer`/`IVertexBuilder`, convert all GlStateManager calls to RenderSystem, follow the section above for calling batched methods from unbatched code, and you're done
3. If you do, start by converting all your pushMatrix/popMatrix/translate/scale/rotate calls to operate on the 
`MatrixStack` instead. Rotations are done by e.g. `GlStateManager.rotate(90, 0, 1, 0)` -> `ms.multiply(Vector3f.POSITIVE_Y.getDegreesQuaternion(90))`.
4. Next, examine the GL calls you make. If it's just "normal" setup that you used to need (i.e. enableStandardItemLighting), and you call other batched vanilla methods after, remove the GL calls and fix the vanilla calls.
5. Finally, if you do custom rendering yourself with the Tessellator, examine the GL calls you make again. See if you can
use one of the layers vanilla defines, otherwise make your own.
6. Then, do `IRenderTypeBuffer.getBuffer(<rendertype>)` and draw into it. The `MatrixStack`'s transforms can be passed
to the bufferbuilder by passing `ms.peek().getModel()` into the first parameter of `vertex()`.

#### Caveat
The current implementation of `IRenderTypeBuffer` does not actually have a batched bufferbuilder for every single render type.
Currently, a number of "popular" render types have batched buffers (for example, the layers for all blocks and items), while all "unrecognized"
layers share a fallback buffer. As soon as a new "unrecognized" layer is seen, the fallback buffer is flushed and drawn and the fallback buffer
is reinitialized with the new layer's state.
It's highly probably that this is just an intermediate step -- eventually all layers may be required to be registered and a buffer made for it, and the fallback buffer removed.

Your code should not care about this implementation detail, as it will change!

#### Other Rendering Miscellanea
* The declared render type (solid, translucent, cutout) for a block has moved from a method override to a registration method. Call `RenderTypeLookup.setRenderLayer` in FMLClientSetupEvent to do this.
* I used to use GlStateManager.color and I can't anymore!
  * The color should no longer be part of this magical implicit global GL state, but part of your actual vertex data.
  * Change your `VertexFormat` to include a color attribute and explicitly pass the color for each vertex.

### Sample Code
Botania's collection of custom `RenderType`s that it needs can be seen [here](https://github.com/Vazkii/Botania/blob/master/src/main/java/vazkii/botania/client/core/helper/RenderHelper.java#L102).

*Sourced from https://gist.github.com/williewillus/30d7e3f775fe93c503bddf054ef3f93e*
