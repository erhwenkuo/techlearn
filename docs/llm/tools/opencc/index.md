# Open Chinese Convert 開放中文轉換

- [OpenCC](https://github.com/BYVoid/OpenCC)
- [opencc-python](https://github.com/yichen0831/opencc-python)

## 介紹

OpenCC 是中文簡繁轉換開源專案，支持詞彙級別的轉換、異體字轉換和地區習慣用詞轉換（中國大陸、臺灣、香港、日本新字體）。不提供普通話與粵語的轉換。

### 特點

- 嚴格區分「一簡對多繁」和「一簡對多異」。
- 完全兼容異體字，可以實現動態替換。
- 嚴格審校一簡對多繁詞條，原則爲「能分則不合」。
- 支持中國大陸、臺灣、香港異體字和地區習慣用詞轉換，如「裏」「裡」、「鼠標」「滑鼠」。
- 詞庫和函數庫完全分離，可以自由修改、導入、擴展。

## 安裝

```bash
pip install opencc
```

## 使用方式

```python
from opencc import OpenCC

cc = OpenCC('s2t')  # 將簡體中文轉換為繁體中文

# 也可以透過呼叫 set_conversion 設定轉換

# cc.set_conversion('s2tw')
to_convert = '开放中文转换'

converted = cc.convert(to_convert)
```

## 配置文件

預設配置文件:

- `s2t.json` Simplified Chinese to Traditional Chinese 簡體到繁體
- `t2s.json` Traditional Chinese to Simplified Chinese 繁體到簡體
- `s2tw.json` Simplified Chinese to Traditional Chinese (Taiwan Standard) 簡體到臺灣正體
- `tw2s.json` Traditional Chinese (Taiwan Standard) to Simplified Chinese 臺灣正體到簡體
- `s2hk.json` Simplified Chinese to Traditional Chinese (Hong Kong variant) 簡體到香港繁體
- `hk2s.json` Traditional Chinese (Hong Kong variant) to Simplified Chinese 香港繁體到簡體
- `s2twp.json` Simplified Chinese to Traditional Chinese (Taiwan Standard) with Taiwanese idiom 簡體到繁體（臺灣正體標準）並轉換爲臺灣常用詞彙
- `tw2sp.json` Traditional Chinese (Taiwan Standard) to Simplified Chinese with Mainland Chinese idiom 繁體（臺灣正體標準）到簡體並轉換爲中國大陸常用詞彙
- `t2tw.json` Traditional Chinese (OpenCC Standard) to Taiwan Standard 繁體（OpenCC 標準）到臺灣正體
- `hk2t.json` Traditional Chinese (Hong Kong variant) to Traditional Chinese 香港繁體到繁體（OpenCC 標準）
- `t2hk.json` Traditional Chinese (OpenCC Standard) to Hong Kong variant 繁體（OpenCC 標準）到香港繁體
- `t2jp.json` Traditional Chinese Characters (Kyūjitai) to New Japanese Kanji (Shinjitai) 繁體（OpenCC 標準，舊字體）到日文新字體
- `jp2t.json` New Japanese Kanji (Shinjitai) to Traditional Chinese Characters (Kyūjitai) 日文新字體到繁體（OpenCC 標準，舊字體）
- `tw2t.json` Traditional Chinese (Taiwan standard) to Traditional Chinese 臺灣正體到繁體（OpenCC 標準）
