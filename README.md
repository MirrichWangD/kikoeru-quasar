# Kikoeru (kikoeru-quasar)

用于收听 DLsite 音色作品的本地网络媒体播放器。

[![unstable build status](https://github.com/umonaca/kikoeru-quasar/actions/workflows/build-and-publish.yml/badge.svg)](https://github.com/umonaca/kikoeru-quasar/actions)

## 前言

本项目是基于以下大佬的开源代码进行二次开发，用于管理本地音色文件夹。在这里特此感谢大佬们用爱发电，并且欢迎各位一起交流学习（本人前端小白）。

- [**umonaca**](https://github.com/umonaca/kikoeru-express)
- [**number178**](https://https://github.com/Number178/kikoeru-express)

外观设计参考某个**asmr**站长，感谢站长多年以来的无私奉献！

## 安装依赖

请先安装全局依赖的quasar，本项目使用的是基于vue 2.x的quasar 1.x-2.0版本。

安装完成后请使用 `quasar -v` 测试是否输出quasar版本以检测安装是否成功

```bash
npm i -g @quasar/cli@2.0.0
# 安装项目依赖（请在本文件夹下运行！）
npm install
```

### 开发者模式启动

该模式下每次修改代码都会重新加载、实时报错等。

```bash
quasar dev
```

使用该模式请同时启动后端并且在浏览器输入`http://<ip>:8080`进入页面，从而达到前后端联调。

### 正式构建模式

- SPA模式构建：

    ```bash
    quasar build
    ```

- PWA模式构建：

    ```bash
    quasar build -m pwa
    ```

构建完成后会输出`dist/`文件夹，请将里面`spa`或`pwa`的文件全部复制到[kikoeru-express](https://github.com/MirrichWangD/kikoeru-express)项目的`dist/`文件夹下，启动项目后可以直接通过`http://<ip>:8888`（默认8888端口）浏览网站。

### 自定义配置

查看文档：[Configuring quasar.conf.js](https://quasar.dev/quasar-cli/quasar-conf-js).
