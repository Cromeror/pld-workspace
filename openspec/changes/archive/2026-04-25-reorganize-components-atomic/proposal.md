# Proposal: Reorganize `pld-web` components under strict Atomic Design

## Intent

`pld-web/src/components/` mixes Atomic Design layers (`atoms/`, `molecules/`, `organisms/`) with feature-grouped folders (`auth/`, `create-users/`, `external-users/`, `reporting-entity/`) and a residual `ui/` shadcn-style primitives folder. The wizard work in commit `c24fa2c` introduced the feature folders without honoring the existing atomic layout, leaving the codebase inconsistent. This change enforces a single convention: classify by atomic level, then group by feature only as a sub-folder inside `molecules/` and `organisms/`.

## Scope

### In Scope (pld-web only)
- Move 52 components into the new layout per the locked reclassification table (D1, D2).
- Promote primitives in `components/ui/` to `components/atoms/` and remove `ui/` entirely.
- Inside `organisms/` and `molecules/`, group by feature (`auth/`, `external-users/`, `reporting-entity/`, `create-users/`, `wizard/`); keep generic ones flat.
- Normalize `ValidationInfoItem.tsx` and `ValidationItem.tsx` (and the inline `BeneficiaryOption.tsx`) to the `Component/index.tsx` folder pattern.
- Rewrite all 28 affected import statements across 6 files.
- Remove now-empty folders: `auth/`, `create-users/`, `external-users/`, `reporting-entity/`, `ui/`.

### Out of Scope
- `pld-api` — no BE work.
- `components/icons/` — stays as-is (idiomatic for SVG components).
- Component refactors, prop API changes, or behavior tweaks of any kind.
- New components or new tests.
- Storybook / docs reorganization.

## Approach

Pure mechanical move + import-path rewrite, no behavior change:

1. Create the new directory skeleton under `pld-web/src/components/{atoms,molecules,organisms}/...`.
2. `git mv` each component to its new location per the reclassification table.
3. For `ValidationInfoItem.tsx`, `ValidationItem.tsx`, and `BeneficiaryOption.tsx`, create `<Component>/index.tsx` and move the file content there.
4. Rewrite the 28 import statements across the 6 known files using a scripted `grep`/`sed` pass on `@/components/` paths (the explore confirmed every import is static — no dynamic `import()` or lazy loads to chase).
5. Remove the 5 emptied folders (`auth/`, `create-users/`, `external-users/`, `reporting-entity/`, `ui/`).
6. Verify with `yarn tsc --noEmit` then `yarn build`.

The `@/` alias from `vite.config.ts` and `tsconfig.app.json` already covers the new paths — no tooling change needed.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `pld-web/src/components/atoms/` | New entries | Receives `Button`, `InputText`, `Checkbox`, `Dropdown`, `RadioButton`, `Calendar`, `TypeItem`, `ValidationInfoItem`. |
| `pld-web/src/components/molecules/` | New entries | Receives `external-users/{PersonTypeAccordion, BeneficiaryOption}`, `create-users/ValidationItem`. |
| `pld-web/src/components/organisms/` | New subtree | New `auth/`, `wizard/`, `external-users/`, `reporting-entity/` sub-folders with their components. |
| `pld-web/src/components/ui/` | Removed | All primitives promoted to `atoms/`. |
| `pld-web/src/components/{auth,create-users,external-users,reporting-entity}/` | Removed | Replaced by atomic layout with feature sub-folders inside the right level. |
| `pld-web/src/components/icons/` | No change | Stays flat, idiomatic SVG components. |
| 6 consumer files (pages, layouts, route configs) | Modified | 28 import paths rewritten under `@/components/`. |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Missed import → `tsc` build break | Med | Explore listed all 28 imports across 6 files; scripted rewrite + `yarn tsc --noEmit` gate before commit. |
| Hidden dynamic / lazy import not caught | Low | Explore confirmed all imports are static; additionally grep for `import\(` on `@/components/` paths during apply. |
| Naming collision after move | Very low | Explore confirmed 0 conflicts. |
| Git history hard to follow after rename | Low | Use `git mv` so history follows; add `--follow` reminder in archive notes. |
| Login flow regression | Very low | Pure path rewrite, no logic change in `LoginForm`; smoke-test `POST /auth/login` from the UI after `yarn build`. |

## Rollback Plan

`git revert` the reorganization commit(s). Because the change is a pure move + import rewrite within a single sub-repo, revert is mechanically clean and produces no schema or runtime state to undo.

## Dependencies

- None. Existing `@/` alias in `vite.config.ts` and `tsconfig.app.json` already supports the new tree.

## Success Criteria

- [ ] `yarn tsc --noEmit` returns 0 errors in `pld-web/`.
- [ ] `yarn build` (tsc -b && vite build) returns 0 errors in `pld-web/`.
- [ ] No folder remains under `pld-web/src/components/` for old feature names (`auth/`, `create-users/`, `external-users/`, `reporting-entity/`) or for `ui/`. `icons/` is preserved.
- [ ] All 28 imports updated; zero remaining matches for `@/components/ui/`, `@/components/auth/`, `@/components/create-users/`, `@/components/external-users/`, `@/components/reporting-entity/` across the repo.
- [ ] `POST /auth/login` from the UI still returns 201 and lands the user post-login (smoke test).
