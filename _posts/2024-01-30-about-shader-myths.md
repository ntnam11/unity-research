## About Shader Myths

One day, I received a question from my co-worker:

> Hey, how do I get 0.2 from 5 without using division (like 1/5)?

It's an interesting question. Turns out he was working with shaders and he wanted to avoid division.

"It's good for performance to avoid division in shaders" - He said.

I've heard of it, but deep in mind I don't know why.
Today he sent me an interesting video: [Investigating and Dispelling Shader Myths... with Science!](https://youtu.be/e5RixckFonU)

The author did the test on Unreal 5.1 with A6000RTX to demystify these myths: (I took everything on his slide, so kudos to him)

> - Instruction Count == Performance
> - Multiply is better than Divide
> - Pow's Cost is Exponential
> - Pack Floats to Vectors for faster math
> - Bigger Textures are more expensive to sample
> - Use LUTs instead of math
> - Use math instead of LUTs

As a Unity developer, I wonder if his conclusion about those myths are the same in Unity and GLES (since I also work on Android platform).

### Test Method

#### Test Specs

- Windows
    - i5-11400F
    - RTX 3060
    - DirectX 11
- Android
    - Pixel 6a with Android 12
    - Mali G78
- Unity
    - 2022.3.17f1
    - Universal RP 14.0.9
    - Both GLES 3 and Vulkan will be tested
- [Android GPU Inspector](https://developer.android.com/agi)
- [Mali Offline Compiler](https://developer.arm.com/Tools%20and%20Software/Mali%20Offline%20Compiler)
- [ARM Streamline](https://developer.arm.com/Tools%20and%20Software/Streamline%20Performance%20Analyzer)
#### Test Input

We will use the default Unity scene, with a default quad using test material.
For the material, we will use customized Unlit shaders.
This scene setup is for avoiding other elements that might affect the game FPS (i.e. lighting, shadows, etc.).

Here's a quick preview of our scene setup:

![Scene setup](/images/2024-01-30-scene-setup.png)

For compiling shaders, besides Unity Shader compiler, I also use [Mali Offline Compiler](https://developer.arm.com/Tools%20and%20Software/Mali%20Offline%20Compiler) to check for additional data

Since I'm using Unity Personal, it will take approx. 6 seconds to wait for the Unity logo to disappear. Therefore I'll skip this period of time, as shown in the below image (taken from [ARM Streamline](https://developer.arm.com/Tools%20and%20Software/Streamline%20Performance%20Analyzer))

![Sample of data](/images/2024-01-30-sample-data.png)

### Hot takes

While I was choosing target frame rate for the test, I found something interesting.

In the image below, left = 30fps, right = 60fps.

![FPS Test sample](/images/2024-01-30-test-fps-streamline.png)

It seems doubling the fps doubles the cost on both CPU & GPU, but in 60fps we see a lower GPU Core Utilization (?). I'll go deeper into this in another post (if possible).

For now I will use 30fps as it provides lower overheads.

### Instruction Count and Performance

***Disclaimer: I'm still a newbie in the shader world, so [Drop me an email](mailto:ntnam117@gmail.com) or [DM me on Discord](https://discordapp.com/users/665767736407883807) if you see something that could possibly be wrong / need improvement :)***

#### Test Description

In this test, I'd like to count the number of instruction and check if it affects performance.
I also want to compare between self-write shader and the one generated with shadergraph (although the result is quite obvious: Shadergraph generates more code for more shader variants, in exchange for more memory and intruction counts).

Here is the sample shader code (following [Writing custom shaders - Unity](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/writing-shaders-urp-basic-unlit-structure.html))

```hlsl
Shader "Unlit/Test_InstructionCount_Shader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white"
        _Scale ("Scale", Float) = 1
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };
            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;
            float _Scale;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the texture
                fixed4 col = mul(tex2D(_MainTex, i.uv), _Scale);
                return col;
            }
            ENDCG
        }
    }
}

```

and a Shader Graph version

![Test Instruction Count - Shadergraph setup](/images/2024-01-30-test-instruction-count-shadergraph.png)

#### Test Result

For an unknown reason, I couldn't create a capture trace in Android GPU Inspector - GLES3 build, so I'll focus on Mali Offline Compiler result and data from ARM Streamline

I wrote a script to quickly check Mali Offline Compiler output in Unity, you can find it [here](https://gist.github.com/ntnam11/268db1d353480b4a3c0035bdeb8aadbb).

##### Mali Offline Compiler result

|Type                  |Description             |   A|  LS|   V|   T|Bound| W| U|
|-                     |-                       |-   |-   |-   |-   |-    |- |- |
|Shader - Vertex       |Total instruction cycles|0.50|2.00|    |0.00|   LS|10|44|
|Shader - Vertex       |Shortest path cycles    |0.50|2.00|    |0.00|   LS|10|44|
|Shader - Vertex       |Longest path cycles     |0.50|2.00|    |0.00|   LS|10|44|
|Shader - Fragment     |Total instruction cycles|0.06|0.00|0.25|0.25| V, T| 5| 4|
|Shader - Fragment     |Shortest path cycles    |0.06|0.00|0.25|0.25| V, T| 5| 4|
|Shader - Fragment     |Longest path cycles     |0.06|0.00|0.25|0.25| V, T| 5| 4|
|Shadergraph - Vertex  |Total instruction cycles|0.44|2.00|    |0.00|   LS| 9|46|
|Shadergraph - Vertex  |Shortest path cycles    |0.44|2.00|    |0.00|   LS| 9|46|
|Shadergraph - Vertex  |Longest path cycles     |0.44|2.00|    |0.00|   LS| 9|46|
|Shadergraph - Fragment|Total instruction cycles|0.08|0.00|0.25|0.25| V, T| 5| 8|
|Shadergraph - Fragment|Shortest path cycles    |0.08|0.00|0.25|0.25| V, T| 5| 8|
|Shadergraph - Fragment|Longest path cycles     |0.08|0.00|0.25|0.25| V, T| 5| 8|

A = Arithmetic, LS = Load/Store, V = Varying, T = Texture, W = Work registers, U = Uniform registers

##### ARM Streamline result



##### Comparison


#### Explanation


### Multiply and Division

