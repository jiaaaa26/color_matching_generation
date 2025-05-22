# 配色生成器 (Color Scheme Generator)

一个简洁、高效的在线配色方案生成工具，帮助设计师和开发者快速创建和谐的配色方案。

## ✨ 功能特色

- 🎨 **多种配色理论**: 支持单色、互补色和三色调和配色方案
- 🔄 **实时调整**: 直观的色相、饱和度、亮度调节滑块
- 💾 **方案保存**: 本地保存您喜欢的配色方案，随时恢复使用
- 📤 **多种导出格式**: 支持CSS变量、PNG图片和GIMP调色板(.gpl)格式导出
- 🌓 **暗色/亮色模式**: 支持手动切换深色/浅色主题
- 📱 **响应式设计**: 完美适配桌面和移动设备
- 🔄 **直观复制**: 点击颜色快速复制十六进制颜色代码

## 🚀 在线演示

访问[配色生成器](https://color-matching-generation.vercel.app)开始使用！(部署后更新链接)

## 🛠️ 技术栈

- **Vue 3**: 前端框架，采用Composition API
- **Vite**: 构建工具和开发服务器
- **Chroma.js**: 强大的颜色处理库
- **localStorage**: 本地保存配色方案
- **Canvas API**: 用于生成PNG导出

## 💻 本地开发

### 前提条件

- Node.js (v14+)
- npm 或 yarn

### 安装步骤

1. 克隆仓库
   ```bash
   git clone https://github.com/your-username/color_matching_generation.git
   cd color_matching_generation
   ```

2. 安装依赖
   ```bash
   npm install
   # 或
   yarn
   ```

3. 启动开发服务器
   ```bash
   npm run dev
   # 或
   yarn dev
   ```

4. 打开浏览器访问 `http://localhost:5173`

### 构建生产版本

```bash
npm run build
# 或
yarn build
```

## 📖 使用指南

1. **选择基础颜色**: 使用颜色选择器或输入十六进制颜色值
2. **浏览配色方案**: 自动生成的单色、互补色和三色调和方案
3. **调整参数**: 使用滑块微调色相、饱和度和亮度
4. **导出方案**: 导出为CSS变量、PNG图像或GIMP调色板
5. **保存喜爱的方案**: 点击"保存方案"按钮将当前配色保存到本地
6. **恢复已保存方案**: 访问"已保存的方案"部分，恢复基础颜色或调整参数

## 🔜 未来计划

- [ ] 添加更多配色理论 (四色方案、分裂互补色)
- [ ] 色彩可访问性检查 (对比度分析)
- [ ] 从图片提取配色方案
- [ ] 导出为Adobe色板格式
- [ ] 配色方案评分系统
- [ ] 用户账户和云端保存
- [ ] 配色社区和分享功能

## 📝 反馈与贡献

欢迎提交问题报告、功能请求或贡献代码！

1. Fork 该仓库
2. 创建你的特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交你的更改 (`git commit -m 'Add some amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 提交 Pull Request


## 👨‍💻 作者

jiaaaa26

## 🙏 致谢

- [Chroma.js](https://github.com/gka/chroma.js) - 用于颜色操作和生成
- [Vue.js团队](https://vuejs.org/) - 出色的前端框架
- 所有提供反馈和建议的用户

---

⭐️ 如果你喜欢这个项目，别忘了给它点个星！
