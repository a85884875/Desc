## 模型与材质 - 自定义shader材质增加阴影

1. ccc 的 3D 材质的阴影渲染
   - 注意点

        1. ccc 只支持 4 个光照系统，和两个光照的阴影渲染

        在引擎的forward-renderer.js文件中头部定义了两个常量分别是光照及光照阴影渲染的最大数量

        ``` js
            const CC_MAX_LIGHTS = 4;
            const CC_MAX_SHADOW_LIGHTS = 2;
        ```
        虽然在代码里会把对应的光照一直添加保存进数组里，但是在 ___cc-lights.inc___ 的引入文件的
        CC_CALC_LIGHTS宏函数里可以看见它只取了固定的4个光照, ___shadow.inc___ 里也有定义最大的可渲染阴影的光照数量

    - 给自己自定义的shader添加阴影

        顶点着色器部分
        ``` glsl
            precision highp float;
            #include <cc-local>
            #include <cc-global>
            #include <skinning>
            #include <input-standard>
            // 重要  start  引入shadow的库
            #include <shadow>
            // 重要  end

            #define CC_USE_TEXTURE CC_USE_ATTRIBUTE_UV0 && USE_DIFFUSE_TEXTURE

            uniform MAIN_TILING {
                vec2 mainTiling;
                vec2 mainOffset;
            };

            #if CC_USE_TEXTURE
                out mediump vec2 v_uv0;
            #endif

            #if CC_USE_ATTRIBUTE_COLOR
                out lowp vec4 v_color;
            #endif

            // 重要  start
            out vec3 v_worldNormal;
            out vec3 v_worldPos;
            out vec3 v_viewDirection;
            // 重要  end

            void main () {
                StandardVertInput In;
                CCVertInput(In);

                vec4 position = In.position;
                
                // 重要  start
                // cc_matWorld 世界坐标系矩阵
                // cc_matWorldIT 是先通过cc_matWorld矩阵求逆，在转置矩阵得出
                v_worldNormal = normalize((cc_matWorldIT * vec4(In.normal, 0)).xyz);
                v_worldPos = (cc_matWorld * position).xyz;
                v_viewDirection = normalize(cc_cameraPos.xyz - v_worldPos);
                // 重要  end

                #if CC_USE_ATTRIBUTE_COLOR
                v_color = In.color;
                #endif

                #if CC_USE_TEXTURE
                v_uv0 = In.uv * mainTiling + mainOffset;
                #endif

                CCShadowInput(v_worldPos);

                gl_Position = cc_matViewProj * cc_matWorld * In.position;
            }
        ```

        片段着色器部分
        ``` glsl
            precision highp float;

            #include <alpha-test>
            #include <texture>
            #include <output>
            #include <cc-global>

            // 需要用到里面的CCPhongShading的函数
            #include <shading-phong>

            uniform UNLIT {
                lowp vec4 diffuseColor;
            };

            in vec3 v_worldNormal;
            in vec3 v_worldPos;
            in vec3 v_viewDirection;

            #if USE_DIFFUSE_TEXTURE
                uniform sampler2D diffuseTexture;
            #endif

            #define CC_USE_TEXTURE CC_USE_ATTRIBUTE_UV0 && USE_DIFFUSE_TEXTURE

            #if CC_USE_ATTRIBUTE_COLOR
                in lowp vec4 v_color;
            #endif

            #if CC_USE_TEXTURE
                in mediump vec2 v_uv0;
            #endif

            vec2 getColorPos() {
                // 
                float xWidth = 15.0;
                float yHeight = 15.0;

                vec2 boxPos = vec2(0.0, 0.0);

                float boxXWidth = 1.0 / xWidth;
                float xIndex = floor(v_uv0.x / boxXWidth); 


                float boxYHeight = 1.0 / yHeight;
                float yIndex = floor(v_uv0.y / boxYHeight);

                boxPos.x = xIndex * boxXWidth;
                boxPos.y = yIndex * boxYHeight;

                // return v_uv0;
                return boxPos;
            }

            void main () {
                vec4 color = diffuseColor;

                #if CC_USE_TEXTURE
                CCTexture(diffuseTexture, v_uv0, color);
                #endif

                #if CC_USE_ATTRIBUTE_COLOR
                color *= v_color;
                #endif

                ALPHA_TEST(color);


                // 获取正确的颜色值坐标点
                vec2 pos = getColorPos();
                
                // 重要 这边是主要添加阴影的代码 start
                PhongSurface s;

                vec4 textureColor = texture(diffuseTexture, pos);
                s.diffuse = textureColor.rgb;
                s.opacity = textureColor.a;
                s.normal = v_worldNormal;
                s.position = v_worldPos;
                s.viewDirection = v_viewDirection;
                s.glossiness = 10.0;

                vec4 tempColor = CCPhongShading(s);
                // 重要 end

                gl_FragColor = CCFragOutput(tempColor);;
                // gl_FragColor = vec4(pos.y, 0.0, 0.0, 1.0);
            }
        ```

   - 引入 shading-phong 库 的目的主要是调用 CCPhongShading 函数
   
     ```
     CCPhongShading 函数中有调用一个 CC_CALC_LIGHTS 的宏定义函数，是来自于cc-lights库
     ```

     这个函数主要作用是添加光照计算
     ```glsl
        #define CC_CALC_LIGHTS(surface, result, lightFunc, ambientFunc) \
            result.diffuse = vec3(0, 0, 0); \
            result.specular = vec3(0, 0, 0); \
            \
            CC_CALC_LIGHT(0, surface, result, lightFunc, ambientFunc) \
            CC_CALC_LIGHT(1, surface, result, lightFunc, ambientFunc) \
            CC_CALC_LIGHT(2, surface, result, lightFunc, ambientFunc) \
            CC_CALC_LIGHT(3, surface, result, lightFunc, ambientFunc)
     ```

     ```glsl
        #define CC_CALC_LIGHT(index, surface, result, lightFunc, ambientFunc) \
            #if CC_NUM_LIGHTS > index \
                #if CC_LIGHT_##index##_TYPE == 3 \
                result.diffuse += ambientFunc(s, cc_lightColor[index]); \
                #else \
                LightInfo info##index; \
                #if CC_LIGHT_##index##_TYPE == 0 \
                    info##index = computeDirectionalLighting(cc_lightDirection[index], cc_lightColor[index]); \
                #elif CC_LIGHT_##index##_TYPE == 1 \
                    info##index = computePointLighting(s.position, cc_lightPositionAndRange[index], cc_lightColor[index]); \
                #elif CC_LIGHT_##index##_TYPE == 2 \
                    info##index = computeSpotLighting(s.position, cc_lightPositionAndRange[index], cc_lightDirection[index], cc_lightColor[index]); \
                #endif \
                \
                Lighting result##index = lightFunc(surface, info##index); \
                CC_CALC_SHADOW(index, result##index) \
                result.diffuse += result##index.diffuse; \
                result.specular += result##index.specular; \
                #endif \
            #endif
     ```
     ```glsl
        ps：
            CC_LIGHT_##index##_TYPE  -> ##index##  -> ##变量名## -> 在C语言的宏定义里叫记号黏贴运算符

            这边每行最后都加一个反斜杠，是因为反斜杠是续行符，因为这一整个函数都是宏定义，宏定义在语法上只能占一行，由于定义结构太长，分行书写，需要用续行符来指明
     ```

   - 链接

     1. <a href=http://c.biancheng.net/view/446.html >宏定义</a>
     1. <a href=https://blog.csdn.net/liaojiacai/article/details/47340927 >续行符</a>
