# 自包含 SSD1306 驱动（免外部库）

Wokwi 的 Library Manager 可能因库索引域名超时而不可用，外部库（Adafruit GFX/SSD1306）装不上。
**对策：项目代码只依赖 Arduino 内置的 `Wire.h`，自写 SSD1306 驱动。** 本文件给出完整可用代码。

## 适用

- Wokwi 上任何 SSD1306（I2C 0x3C）项目，包括 ESP32-C3 / ESP32 / Arduino 等。
- 接线：SCL、SDA 接开发板 I2C 引脚（ESP32-C3: SDA=5, SCL=6）；VCC=3V3，GND。

## 驱动原理

- SSD1306 是 I2C 从机：控制字节 `0x00`（命令）/ `0x40`（数据）。
- 显存 128×64 = 8 页（page）× 128 列；写显存时按页寻址（`0xB0+p`），每页连续写 128 字节。
- 初始化命令序列固定（见 `initO()`）；帧缓冲 `uint8_t fb[1024]`，`show()` 全量刷屏。

## 完整代码（已验证，编译通过、仿真运行）

```cpp
#include <Wire.h>
#define W 128
#define H 64
#define A 0x3C
uint8_t fb[1024];
void cmd(uint8_t c){Wire.beginTransmission(A);Wire.write(0x00);Wire.write(c);Wire.endTransmission();}
void initO(){Wire.begin(5,6);cmd(0xAE);cmd(0x20);cmd(0x00);cmd(0xB0);cmd(0xC8);cmd(0x00);cmd(0x10);cmd(0x40);cmd(0x81);cmd(0xCF);cmd(0xA1);cmd(0xA6);cmd(0xA8);cmd(0x3F);cmd(0xA4);cmd(0xD3);cmd(0x00);cmd(0xD5);cmd(0xF0);cmd(0xD9);cmd(0x22);cmd(0xDA);cmd(0x12);cmd(0xDB);cmd(0x40);cmd(0x8D);cmd(0x14);cmd(0xAF);}
void show(){for(int p=0;p<8;p++){cmd(0xB0+p);cmd(0x00);cmd(0x10);Wire.beginTransmission(A);Wire.write(0x40);for(int i=0;i<W;i++)Wire.write(fb[p*W+i]);Wire.endTransmission();}}
void px(int x,int y,int on){if(x<0||x>=W||y<0||y>=H)return;int i=y/8*W+x;if(on)fb[i]|=1<<(y%8);else fb[i]&=~(1<<(y%8));}
void cir(int cx,int cy,int r,int on){for(int y=cy-r;y<=cy+r;y++)for(int x=cx-r;x<=cx+r;x++){int dx=x-cx,dy=y-cy;if(dx*dx+dy*dy<=r*r)px(x,y,on);}}
void rect(int x,int y,int w,int h,int on){for(int j=y;j<y+h;j++)for(int i=x;i<x+w;i++)px(i,j,on);}
void line(int x0,int x1,int y,int on){for(int x=x0;x<=x1;x++)px(x,y,on);}
```

## 眨眼示例（loop 骨架）

```cpp
unsigned long lt=0,t=0,p=0;float b=0,gx=0,gy=0,tx=0,ty=0;bool bl=false;
void setup(){initO();randomSeed(analogRead(0));tx=(float)random(-100,101)/100;ty=(float)random(-70,71)/100;gx=tx;gy=ty;}
void drawEye(int cx,int cy,int lid){
  cir(cx,cy,14,1);
  if(lid<24){int qx=cx+(int)(gx*8),qy=cy+(int)(gy*8);cir(qx,qy,6,0);cir(qx-2,qy-2,2,1);}
  if(lid>=28)line(cx-14,cx+14,cy,1);
  else if(lid>0){rect(cx-14,cy-14,28,lid,0);line(cx-14,cx+14,cy-14+lid-1,1);}
}
void loop(){
  unsigned long n=millis();float dt=min((n-lt)/1000.0f,0.1f);lt=n;
  if(n-p>random(1800,4200)){p=n;tx=(float)random(-100,101)/100;ty=(float)random(-70,71)/100;}
  float k=min(1.0f,dt*6);gx+=(tx-gx)*k;gy+=(ty-gy)*k;
  float lid=0;
  if(!bl&&n-t>(unsigned long)random(2500,5200)){bl=true;b=0;t=n;}
  if(bl){b+=9*dt;if(b>=2){bl=false;t=n;}lid=(b<=1)?b:(2-b);}
  memset(fb,0,1024);
  drawEye(46,28,(int)(lid*28));
  drawEye(82,28,(int)(lid*28));
  show();delay(10);
}
```

## diagram.json 模板（ESP32-C3 + SSD1306，走线优化版）

```json
{
  "version": 1,
  "author": "doubao",
  "editor": "wokwi",
  "parts": [
    { "type": "board-esp32-c3-devkitm-1", "id": "esp", "top": 0, "left": 0, "attrs": {} },
    { "type": "board-ssd1306", "id": "oled1", "top": 150, "left": 260, "attrs": { "i2cAddress": "0x3c" } }
  ],
  "connections": [
    [ "esp:5", "oled1:SDA", "green", [ "h-40", "v-25", "*", "v20", "h-10" ] ],
    [ "esp:6", "oled1:SCL", "blue", [ "h-40", "v-50", "*", "v20", "h-10" ] ],
    [ "esp:3V3.1", "oled1:VCC", "red", [ "h-40", "v-75", "*", "v20", "h-10" ] ],
    [ "esp:GND.1", "oled1:GND", "black", [ "h-40", "v-100", "*", "v20", "h-10" ] ]
  ]
}
```

## 走线美化（Wire Routing）

Wokwi 的线**永远横平竖直**（水平或垂直，绝不斜线），美观与否由 `connections` 第 4 个元素的路径指令决定。

### 路径指令语法（官方 mini-language）

- `"v<数字>"`：垂直移动。`v10` 向下 10px，`v-10` 向上 10px
- `"h<数字>"`：水平移动。`h10` 向右 10px，`h-10` 向左 10px
- `"*"`：最多出现一次，把指令分成两段——`*` 前作用于**源引脚**，`*` 后作用于**目标引脚**（从目标端反向应用：列表最后一个先执行）
- 例：`["v10", "h5", "*", "v-15", "h10"]` → 源引脚下 10 再右 5；目标引脚右 10 再上 15；剩余距离由仿真器自动补横竖线
- 支持小数（如 `h134.4`、`v27.6`）

### 美观设计原则

1. **源端先沿引脚方向出线（关键，避免与引脚重叠）**：引脚朝左（如 ESP32-C3 左列引脚）就先 `h-` 水平离开板体（如 `h-40`），**再** `v` 上下错位；绝不要先 `v` 垂直引——竖段会穿过同列的其他引脚
2. **源端瀑布错位**：多条线在板体外空区用 `v` 值递增错开（如 `v-25/50/75/100`），让水平段分布在**不同高度**，互不重叠
3. **目标端平行引出**：目标端用统一指令（如 `*` 后 `v20, h-10`），利用引脚天然间距形成平行总线感；引脚朝下（如 OLED 底部）用 `v` 正数向下引出，避免线贴进屏体
4. **颜色语义**：SDA=green、SCL=blue、VCC=red、GND=black（与实物线色一致）
5. **布局先行**：器件 `top/left` 决定走线空间——外设放主控右下方、引脚朝外，走线最短最整齐；两器件间预留空区作为走线走廊
6. **无交叉检查**：每条线的竖直段只在板体外的空区移动，水平段高度彼此不同 → 无交叉

### 微调方法

- 观感依赖器件实际布局，渲染后如某条线拐得不好看：调该线的 `v`/`h` 数值（小步 5-10px）
- 也可以直接在 Wokwi 页面上**拖动物件/引脚**微调，改动会自动写回 diagram.json
- 验证走线：`wokwi-cli lint` 检查非法连接；渲染效果用浏览器截图确认

## 要点

- `Wire.begin(5,6)` 必须与 diagram 接线一致（SDA=5、SCL=6）。
- 不要在任何文件里 `#include <Adafruit_*>`，否则 Wokwi 会尝试下载外部库并可能失败。
- 如需更大刷新率，可只更新脏页；当前全量刷屏在仿真中足够。
