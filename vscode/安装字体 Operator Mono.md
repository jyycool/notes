# 安装字体 Operator Mono

在 [下载页](https://github.com/beichensky/Font) 上下载该项目到本地.

### 1. 安装 Fira 字体

将上述下载包下路径 `FiraCode/ttf` 中的文件全部安装

### 2. 安装 Operator Mono 字体

将上述下载包下路径 `Operator Mono` 中的文件全部安装

### 3. 配置 Vscode

在 `settings.json` 添加如下配置

```json
"editor.fontFamily": "Operator Mono",
"editor.fontLigatures": true, // 这个控制是否启用字体连字，true启用，false不启用，这里选择启用
"editor.tokenColorCustomizations": {
	"textMateRules": [
		{
			"name": "italic font",
			"scope": [
				"comment",
        "keyword",
        "storage",
        "keyword.control.import",
        "keyword.control.default",
        "keyword.control.from",
        "keyword.operator.new",
        "keyword.control.export",
        "keyword.control.flow",
        "storage.type.class",
        "storage.type.function",
        "storage.type",
        "storage.type.class",
        "variable.language",
        "variable.language.super",
        "variable.language.this",
        "meta.class",
        "meta.var.expr",
        "constant.language.null",
        "support.type.primitive",
        "entity.name.method.js",
        "entity.other.attribute-name",
        "punctuation.definition.comment",
        "text.html.basic entity.other.attribute-name.html",
        "text.html.basic entity.other.attribute-name",
        "tag.decorator.js entity.name.tag.js",
        "tag.decorator.js punctuation.definition.tag.js",
        "source.js constant.other.object.key.js string.unquoted.label.js",
			],
			"settings": {
				"fontStyle": "italic",
			}
		},
	]
},
"editor.fontSize": 14
```

