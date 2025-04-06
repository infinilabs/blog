# INFINI Labs Blog

Official blog repository for INFINI Labs, built with Hugo.

## Quick Start

### Prerequisites

- Node.js (v18 or higher)
- Hugo (v0.145.0 or higher)
- pnpm (latest version)

### Local Development

1. Clone the repository

```bash
git clone https://github.com/infinilabs/blog.git

cd blog
```

2. Install dependencies

```bash
npm install -g pnpm

pnpm install
```

3. Start development server

```bash
pnpm dev
```

Visit http://localhost:1313 to view the blog.

### Creating New Posts

Create a new post using:

```bash
hugo new content/chinese/posts/your-post-name.md
# or for English posts
hugo new content/english/posts/your-post-name.md
```

### Building

Build for production:

```bash
pnpm build
```

## Project Structure

```
.
├── content/          # Blog content
│   └── english/     # English articles
├── assets/          # Source assets (images, SCSS, JS, etc.)
├── static/          # Static assets
├── themes/          # Theme files
└── config.toml      # Hugo configuration
```

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details

## Contact

- Website: https://infinilabs.com
- GitHub: https://github.com/infinilabs

🎉 🎉 Exciting news from INFINI Labs! We've officially open-sourced our products on GitHub. 👉 Check it out here: http://github.com/infinilabs