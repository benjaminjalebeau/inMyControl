# Contributing to InMyControl

Thank you for your interest in contributing to InMyControl! This document provides setup instructions, development guidelines, and project conventions.

## Development Setup

### Prerequisites

- Node.js 18+ and npm/yarn/pnpm
- Git
- Supabase account (free tier works)
- Vercel account (for deployment)
- Google Cloud Console project (for Google Calendar integration)
- Code editor (VS Code recommended)

### Initial Setup

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd InMyControl
   ```

2. **Install dependencies:**
   ```bash
   npm install
   # or
   yarn install
   # or
   pnpm install
   ```

3. **Set up environment variables:**
   
   Create a `.env.local` file in the root directory:
   ```env
   # Supabase
   NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
   NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
   SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

   # Google Calendar (optional for MVP)
   GOOGLE_CLIENT_ID=your_google_client_id
   GOOGLE_CLIENT_SECRET=your_google_client_secret

   # Sentry (optional)
   SENTRY_DSN=your_sentry_dsn

   # App URL
   NEXT_PUBLIC_APP_URL=http://localhost:3000
   ```

4. **Set up Supabase:**
   - Create a new Supabase project at https://supabase.com
   - Run the database migrations from [`DATABASE.md`](DATABASE.md)
   - Configure Storage buckets: `profile-images`, `attachments`
   - Set up RLS policies as documented

5. **Run the development server:**
   ```bash
   npm run dev
   # or
   yarn dev
   # or
   pnpm dev
   ```

   Open [http://localhost:3000](http://localhost:3000) in your browser.

## Project Structure

```
InMyControl/
├── .github/           # GitHub workflows and templates
├── public/            # Static assets
├── src/
│   ├── app/           # Next.js app directory (if using App Router)
│   ├── components/    # React components
│   ├── lib/           # Utilities and helpers
│   ├── hooks/         # Custom React hooks
│   ├── types/         # TypeScript type definitions
│   ├── api/           # API routes (serverless functions)
│   └── styles/        # CSS/styles
├── supabase/          # Supabase migrations and functions
├── docs/              # Additional documentation
├── tests/             # Test files
└── README.md
```

## Development Guidelines

### Code Style

- Use TypeScript for all new code
- Follow ESLint and Prettier configurations
- Use functional components with hooks
- Prefer named exports over default exports for components
- Use meaningful variable and function names

### Git Workflow

1. **Create a feature branch:**
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Make your changes:**
   - Write clear commit messages
   - Keep commits focused and atomic
   - Test your changes locally

3. **Before pushing:**
   - Run linter: `npm run lint`
   - Run tests: `npm test`
   - Format code: `npm run format` (if configured)

4. **Push and create a Pull Request:**
   ```bash
   git push origin feature/your-feature-name
   ```

5. **PR Checklist:**
   - [ ] Code follows project conventions
   - [ ] Tests pass
   - [ ] Documentation updated if needed
   - [ ] No console errors or warnings
   - [ ] Responsive design tested

### Component Guidelines

- Keep components small and focused
- Use TypeScript interfaces for props
- Extract reusable logic into custom hooks
- Prefer composition over inheritance
- Use Supabase client from context/provider

### Database Guidelines

- Always use parameterized queries
- Rely on RLS policies for security (never bypass)
- Use transactions for multi-step operations
- Add indexes for frequently queried columns
- Document any new database functions

### API Guidelines

- Use Supabase client directly for CRUD when possible
- Use serverless functions for privileged operations
- Always validate authentication
- Return consistent error formats
- Include request/response examples in code comments

### Testing

- Write unit tests for utility functions
- Write integration tests for API endpoints
- Write E2E tests for critical user flows
- Aim for >80% code coverage

## Sprint Workflow

Refer to the Roadmap in [`README.md`](../README.md) for current sprint goals.

1. **Sprint Planning:**
   - Review sprint objectives
   - Assign tasks
   - Set up development environment

2. **Daily Development:**
   - Work on assigned tasks
   - Create feature branches
   - Write tests alongside code
   - Submit PRs early for feedback

3. **Sprint Review:**
   - Demo completed features
   - Gather feedback
   - Update documentation

## Common Tasks

### Adding a New Feature

1. Create a feature branch
2. Implement the feature
3. Add tests
4. Update documentation
5. Submit PR

### Adding a Database Migration

1. Create migration file in `supabase/migrations/`
2. Run migration locally
3. Test RLS policies
4. Document in [`DATABASE.md`](DATABASE.md) if significant
5. Include in PR description

### Adding a New API Endpoint

1. Create endpoint file in `src/api/` or `pages/api/`
2. Add authentication middleware
3. Implement error handling
4. Write tests
5. Update [`API.md`](API.md) documentation

### Adding a New Component

1. Create component file in `src/components/`
2. Add TypeScript types
3. Add Storybook story (if configured)
4. Write component tests
5. Document props and usage

## Debugging

### Local Debugging

- Use browser DevTools for frontend issues
- Check Supabase logs in dashboard
- Review Vercel function logs locally with `vercel dev`
- Use console.log or debugger statements (remove before commit)

### Database Debugging

- Use Supabase SQL Editor to test queries
- Check RLS policy execution in logs
- Verify indexes are being used
- Test with different user contexts

### Common Issues

**RLS blocking queries:**
- Verify user is authenticated: `auth.uid()`
- Check RLS policy conditions
- Test policy in Supabase dashboard

**Storage upload failures:**
- Verify bucket exists and policies are set
- Check file size limits
- Verify user has INSERT permission

**Google Calendar OAuth errors:**
- Verify credentials in environment
- Check OAuth consent screen configuration
- Ensure redirect URIs are configured

## Code Review Process

1. **Submit PR:**
   - Clear title and description
   - Reference related issues
   - Include screenshots for UI changes

2. **Review Checklist:**
   - Code quality and style
   - Test coverage
   - Security considerations
   - Performance implications
   - Documentation updates

3. **Address Feedback:**
   - Respond to comments
   - Make requested changes
   - Re-request review when ready

4. **Merge:**
   - Wait for approval
   - Ensure CI passes
   - Squash commits if requested
   - Delete feature branch after merge

## Documentation

- Update [`README.md`](../README.md) for major features
- Update [`API.md`](API.md) for API changes
- Update [`DATABASE.md`](DATABASE.md) for schema changes
- Add inline code comments for complex logic
- Update this file if workflow changes

## Getting Help

- Check existing documentation first
- Search GitHub issues for similar problems
- Ask questions in PR comments
- Reach out to maintainers if blocked

## License

By contributing, you agree that your contributions will be licensed under the same license as the project.

