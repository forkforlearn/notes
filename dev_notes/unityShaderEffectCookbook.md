
# Unity Shaders and Effects Cookbook

# 1 Diffuse Shading

## Creating a basic Surface Shader

 - Getting Ready ...
     - create a `Materials` folder
     - create a shader, create a material
     - edit material, choose the shader you created
     - apply material to an object

 - The Surface Shader language is a more component-based way of writing Shaders. 
    - Tasks such as processing your own texture coordinates and transformation matrices have already been done for you
 - See also 
    - For more information on where to  nd a large portion of the built-in Cg functions for Unity
    - go to your Unity install directory and navigate to 
        - `CGIncludes\UnityCG.cginc`
        - `CGIncludes\Lighting.cginc`
        - `CGIncludes\UnityShaderVariables.cginc`

## Adding properties to a Surface Shader

```
Properties {
    _Color ("Color", Color) = (1,1,1,1)
}
```

 - from left to right, they are 
    - variable name
    - inspector GUI name
    - type 
    - Default value

---

Surface Shader property types | ·
 --- | ---
Range (min, max) | float property as a slider
Color | a color picker = (float,float,float,float)
2D  |   a texture
Rect |  a non-power-of-2 texture , functions the same as the 2D GUI element
Cube  |  a cube map 
Float |  a float value
Vector |  four-float property

 - See also
    - https://docs.unity3d.com/Manual/SL-Properties.html

---

Property attributes

attributes | /
 --- | ---
[HideInInspector] | does not show the property value in the material inspector.
[NoScaleOffset] | material inspector will not show texture tiling/offset fields for texture properties with this attribute.
[Normal] | indicates that a texture property expects a normal-map.
[HDR] | indicates that a texture property expects a high-dynamic range (HDR) texture.
[Gamma] | indicates that a float/vector property is specified as sRGB value in the UI (just like colors are), and possibly needs conversion according to color space used. See Properties in Shader Programs.
[PerRendererData] | indicates that a texture property will be coming from per-renderer data in the form of a MaterialPropertyBlock. Material inspector changes the texture slot UI for these properties.




## Using properties in a Surface Shader

 1. add a property
    - `_AmbientColor ("Ambient Color", Range(0,10)) = 2`
 2. declare the properties variable type inside of the CGPROGRAM, so we can access its value from the properties block
    - in SubShader{} block , define subshader variables
    - `float4 _AmbientColor;`
 3. using it the `surf()` function
    - `float4 c =  pow((_EmissiveColor + _AmbientColor), _MySliderValue);`

## Creating a custom diffuse lighting model

 1. modify the #pragma statement to
    - `#pragma surface surf BasicDiffuse`  光照模型改成自定义的 `BasicDiffuse`
 2. Add the following code to the subshader:
 3. Save the Shader and return to Unity.
    - you will see that no real visible change has happened to our Material
    - What we have done is removed the connection to the built-in Unity diffuse lighting and created our own lighting model that we can customize.

```
inline float4 LightingBasicDiffuse (SurfaceOutput s, fixed3
        lightDir, fixed atten)
{
    float difLight = max(0, dot (s.Normal, lightDir));  // saturate()
    float4 col;
    col.rgb = s.Albedo * _LightColor0.rgb * (difLight * atten * 2);
    col.a = s.Alpha;
    return col;
}
```

How it works ?

 - The #pragma surface directive tells the Shader which lighting model to use for its calculation
    - when we first create the Shader, the light modle which use is defined in `Lighting.cginc` file
    - We have now told the Shader to look for a lighting model by the name **BasicDiffuse**
 - Creating a new lighting model is done by declaring a new lighting model function. There are 3 types of lighting model functions that you can use:   
    - `half4 Lighting<Your Chosen Name> (SurfaceOutput s, half3 lightDir, half atten){}`
        - This function is used for forward rendering when the view direction is not needed.
    - `half4 Lighting<Your Chosen Name> (SurfaceOutput s, half3 lightDir, half3 viewDir, half atten){}`
        - This function is used in forward rendering when a view direction is needed
    - `half4 Lighting<Your Chosen Name>_PrePass (SurfaceOutput s, half4 light){}`
        - This function is used when you are using deferred rendering for your project
 - To complete the diffuse calculation, we need to multiply `difLight` with 
    - the s.Albedo value (which comes from our surf function) 
    - with the incoming `_LightColor0.rgb` value (which Unity provides)
    - and then multiply the result of that with (difLight * atten) 
 - finally, return that value as the color. See the following code:


[rending path 介绍](https://github.com/mebusy/notes/blob/master/dev_notes/renderingPath.md)


## Creating a Half Lambert lighting model

 - Half Lambert was a technique created by Valve as a way of getting the lighting to show the surface of an object in low-light areas
 - It basically brightens up the diffuse lighting of the Material and wraps the diffuse light around an object's surface
 - It is designed to prevent the rear of an object losing its shape and looking too flat. 


![](https://raw.githubusercontent.com/mebusy/notes/master/imgs/u3d_shader_half_lambert_01.png)

> Alyx model showing Lambertian lighting (left) and Half Lambert lighting (right)

![](https://raw.githubusercontent.com/mebusy/notes/master/imgs/u3d_shader_half_lambert_02.png)

> The Lambertian dot product lobe (red) vs. the Half Lambertian dot product lobe (blue)

 - Using the basic Shader that we created in the last recipe, let's update the diffuse calculation:

```
    float difLight = dot (s.Normal, lightDir);
    float HLAMBERT = difLight*0.5 + 0.5 ;

    float4 col;
    col.rgb = s.Albedo * _LightColor0.rgb * ( HLAMBERT * atten * 2);
    col.a = s.Alpha;
    return col;
```

 - NO more `max` calcuation for difLight 
 - 原来的 0-1 的值，现在变成了 0.5-1, 整体增强了。


## Creating a ramp texture to control diffuse shading

 - Another great tool in your Shader writing toolbox is the use of a ramp texture to drive the color of the diffuse lighting.
 - This allows you to accentuate the surface's colors to fake the effects of more bounce light or a more advanced lighting setup. 
 - You see this technique used a lot more for cartoony games
    - where you need a more artist-driven look to your Shaders and not so much of a physically-accurate lighting model.
 
 ![](http://www.68idc.cn/help/uploads/allimg/150307/1504334Q2_0.jpg)

 - To get this started you will need to create a ramp texture
    - any image editing application should be able to make a gradient
    - ![](https://raw.githubusercontent.com/mebusy/notes/master/imgs/u3d_shader_ramp_texture.png)
 - Simply modify the lighting function so that it includes this new code `ramp` 

```
_RampTex("Ramp texture", 2D) = "white" {}
...
sampler2D _RampTex;
...

    float difLight = dot (s.Normal, lightDir); 
    float hLambert = difLight*0.5 + 0.5 ;
    float3 RAMP = tex2D( _RampTex ,float2(hLambert,hLambert) ).rgb ;

    float4 col;
    col.rgb = s.Albedo * _LightColor0.rgb * ( RAMP );
    col.a = s.Alpha;
    return col;
```

 - tex2D function takes two arguments
    - texture property
    - UV
 - 在一般的Blinn/Phong模型中，我们对漫反射的系数是基于入射光和击中的点的法线的角度
    - 使用Ramp texture，可以用一张1D的材质图来做为索引表。用来控制漫反射的系数
 - Now it is possible for the artist to have some custom control over how the light looks on the surface of an object.
    - This is why this technique is more commonly seen on a project where you need more of an illustrative look.
 - ![](https://raw.githubusercontent.com/mebusy/notes/master/imgs/ramEffect.jpg)

## Creating a faked BRDF using a 2D ramp texture

 - We can take the ramp diffuse recipe one step further by using the `view direction` , provided by the lighting functions, to create a more advanced visual look to our lighting.
 - By utilizing the view direction, we will be able to generate some faked `rim lighting` (边缘光照) .
 - If we look at the ramp diffuse technique, we are only using one value to place into the UV lookup of the ramp texture. 
    - This means that we will get a very linear type of lighting effect. 
    - The view vector will provide us with the means to create a more advance texture lookup.
 - In the Cg industry this technique is often referred to as a **BRDF effect**. 
    - BRDF（Bidirectional Reflectance Distribution Function，即双向反射分布函数）
 - BRDF simply means the way in which light is refected off an opaque surface from both the view direction and the light direction.
 - We need gradients for both dimensions of the texture , and apply it to our Ram texture 
    - ![](https://raw.githubusercontent.com/mebusy/notes/master/imgs/BRDF.jpg)
 - How to do it ...
    1. First we need to change our lighting function to include the `viewDir` variable that Unity provides us
        - to get the current view direction  , modify your lighting function to 
        - `half4 Lighting<Your Chosen Name> (SurfaceOutput s, half3 lightDir, half3 viewDir, half atten){}`
    2. We then need to calculate the dot product of the view direction and the surface normal
        - `float rimLight = dot (s.Normal , viewDir);`
        - This will produce a falloff type effect that we can use to drive our BRDF texture.
    3. To complete the operation, we need to feed our dot product result into the float2() function of the tex2D() function , modify your code to
        - `float3 ramp = tex2D( _RampTex , float2(hLambert,rimLight) ).rgb ;`

---

```
inline float4 LightingBasicDiffuse (SurfaceOutput s, half3 lightDir, half3 viewDir, half atten) { // 1
    float difLight = dot (s.Normal, lightDir); 
    float RIMLIGHT = dot (s.Normal , viewDir); // 2
    float hLambert = difLight*0.5 + 0.5 ;
    float3 ramp = tex2D( _RampTex ,float2(hLambert, RIMLIGHT ) ).rgb ; //3

    float4 col;
    col.rgb = s.Albedo * _LightColor0.rgb * ( ramp );
    col.a = s.Alpha;
    return col;
}
```

![](https://raw.githubusercontent.com/mebusy/notes/master/imgs/BRDF_effect.jpg)

![](https://raw.githubusercontent.com/mebusy/notes/master/imgs/BRDF_explain.jpg)

 - When using the view direction parameter, we can create a very simple falloff type effect. 
 - You can use this parameter to create a lot of different types of effects: 
    - a bubble type transparency
    - the start of a rim light effect
    - shield effects
    - the start of a toon outline effect

-------

# 2 Using Textures for Effects

We can also use texture to animate, to blend and really, to drive any other property we want. 

 - Scrolling textures by modifying UV values
 - Animating sprite sheets
 - Packing and blending textures
 - Normal mapping
 - Creating procedural textures in the Unity editor
 - Photoshop levels effect

## Scrolling textures by modifying UV values

 - One of the most common texture techniques used in today's game industry is 
    - the process of allowing you to scroll their textures over the surface of an object.
    - This allows you to create effects such as **waterfalls, rivers, lava flows, and so on**.
    - It's also a technique that is the basis for creating animated sprite effects.
 - create a new shader and a new matrial
 - 1. The Shader will need two new properties that will allow us to control the speed of the texture scrolling

```
Properties {
    _MainTint ("Diffuse Tint", Color) = (1,1,1,1)
    _MainTex ("Base (RGB)", 2D) = "white" {}
    _ScrollXSpeed( "X Scroll Speed",Range(0,10) ) =2
    _ScrollYSpeed( "Y Scroll Speed",Range(0,10) ) =2
}
...
sampler2D _MainTex;
fixed4 _MainTint ;
fixed _ScrollXSpeed ;
fixed _ScrollYSpeed ;
```

 - 2. Modify the surface function to change the UVs being given to the tex2D() function.
    - hen use the built-in `_Time` variable to animate the UVs over time when **Play** is pressed in the editor:
    - `scrolledUV` has to be float2 / fixed2 because the UV values are being passed to us from the Input structure:
        - `struct Input { float2 uv_Maintex };`

```
void surf (Input IN, inout SurfaceOutputStandard o) {
    // create a separate variable to store our uvs
    // before we pass them to the tex2D() function
    fixed2 scrolledUV = IN.uv_MainTex;

    // create variables that store the 
    // individual x and y 
    // components for uvs scaled by time
    fixed xScrollValue = _ScrollXSpeed * _Time;
    fixed yScrollValue = _ScrollYSpeed * _Time;

    // apply the final uv offset
    scrolledUV += fixed2(xScrollValue,yScrollValue);

    // apply textures and tint
    half4 c = tex2D(_MainTex, scrolledUV);
    o.Albedo = c.rgb * _MainTint ;
    o.Alpha = c.a ;
}
```
 
 - for more shadow lab builtin values : [SL-BuiltinValues](https://docs.unity3d.com/455/Documentation/Manual/SL-BuiltinValues.html)

## Animating sprite sheets

 - create a new shader and a new matrial
    - Then set up your Material by placing it onto a plane in the Scene view
 - 1. Create new properties 

```
Properties {
    _MainTex ("Base (RGB)", 2D) = "white" {}
    _TexWidth ("Sheet Width" , float) = 747
    _CellAmount ("Cell Amount" , float) = 9
    _Speed ( "Speed" , Range(0.01,32) ) = 12
}
...
sampler2D _MainTex;
float _TexWidth ;
float _CellAmount ;
float _Speed ;

```
 
 - get it animated !

```
void surf (Input IN, inout SurfaceOutputStandard o) {
    // let's store our UVs in a separate variable
    fixed2 spriteUV = IN.uv_MainTex;

    // uv 缩放，映射到第一帧
    float UVx = IN.uv_MainTex.x / _CellAmount ;

    float cellUVPercentage = 1.0 / _CellAmount ;

    // let's get a stair step value out of time 
    // so we can increment the uv offset
    float frameIdx = fmod( _Time.y * _Speed , _CellAmount ) ; 
    frameIdx = ceil(frameIdx) ;

    UVx +=  cellUVPercentage* frameIdx ;

    spriteUV = float2( UVx,  spriteUV.y ) ;

    // apply textures and tint
    half4 c = tex2D(_MainTex, spriteUV);
    o.Albedo = c.rgb   ;
    o.Alpha = c.a ;
}
```

 - 注意，我们这里使用 `_Time.y` , 这是正常的time值，如果直接使用 `_Time`, 其实是 `_Time.x`, 它是 t/20.
 - There's more...
    - you can move the frame offset selection code to a C# script that talks to the Shader, and have the CPU drive that portion of the code.

```
// C# 脚本直接修改 shader uniforms
void FixedUpdate() {
    timeValue = .... ;
    tranform.renderer.material.SetFloat("_TimeValue",timeValue);
}
```



