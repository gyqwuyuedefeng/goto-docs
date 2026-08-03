# PlotSidePanel Accordion Behavior Design

## Context

`goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue` renders the plot side panel with Element UI `el-collapse`. The current `activeGroup` state is an array, so multiple collapse items can stay open at the same time. The requested behavior is: when the user opens one tab/collapse item, all other items should close automatically.

The side panel includes a special "上传文件" item with name `-1`, plus visible parameter groups from `plot.userDraw.paramSettingsList`. The new behavior applies to all of these items.

## Goal

The side panel should behave like an accordion:

- Opening "上传文件" closes all parameter groups.
- Opening a parameter group closes "上传文件" and all other parameter groups.
- Clicking the currently open item may leave no item open, matching Element UI accordion behavior.

## Recommended Approach

Use Element UI's native `el-collapse` accordion mode.

Implementation should:

- Add the `accordion` prop to the `el-collapse` in `PlotSidePanel.vue`.
- Change `activeGroup` from an array to a single active collapse name.
- Initialize `activeGroup` to `-1` so "上传文件" is open by default.
- Keep the existing upload, delete, parameter update, permission, and visibility logic unchanged.

`defaultExpandAll` currently conflicts with accordion behavior because accordion mode can only show one item at a time. For this feature, accordion behavior takes precedence. If `defaultExpandAll` is true, initialization should still choose only one item instead of opening every visible item.

## Data Flow

`el-collapse` owns the user interaction and writes the active item name into `activeGroup` through `v-model`. Because `accordion` is enabled, Element UI ensures only one item name is active at any time. Existing child components and emitted events do not need API changes.

## Error Handling

No new async paths or recoverable errors are introduced. Existing file upload and delete error handling remains unchanged.

## Testing

Manual verification is sufficient for this narrow UI behavior:

1. Open the plot draw page.
2. Confirm "上传文件" is open by default.
3. Click a parameter group and confirm "上传文件" closes.
4. Click another parameter group and confirm the previous group closes.
5. Click "上传文件" and confirm the parameter group closes.
6. Click the currently open item again and confirm no other item opens unexpectedly.
