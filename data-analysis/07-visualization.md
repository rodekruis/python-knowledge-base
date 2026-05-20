# 07 — Visualization

## Set Publication-Quality Defaults at the Start

Don't rely on matplotlib's ugly defaults. Set them once at the beginning of the visualization section, not in every plotting cell.

### ✅ Good: set rcParams once

```python
# === Section 8: Visualization ===
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
import cartopy.crs as ccrs
import cartopy.feature as cfeature

plt.rcParams.update({
    'font.size': 11,
    'axes.titlesize': 13,
    'axes.labelsize': 11,
    'xtick.labelsize': 9,
    'ytick.labelsize': 9,
    'figure.dpi': 150,
    'savefig.dpi': 300,
    'savefig.bbox': 'tight',
    'figure.figsize': (10, 6),
    'axes.grid': False,
    'lines.linewidth': 1.5,
})

VIZ_OUTPUT = OUTPUT_DIR / "figures"
VIZ_OUTPUT.mkdir(parents=True, exist_ok=True)
print(f"✓ Visualization settings configured, saving to: {VIZ_OUTPUT}")
```

### ❌ Bad: inconsistent formatting across cells

```python
# Cell 1
plt.figure(figsize=(12, 8))
plt.title("Plot A", fontsize=14)

# Cell 5  
plt.figure(figsize=(8, 5))
plt.title("Plot B", fontsize=10)  # Different size — looks inconsistent
```

## One Plot Per Cell

Each cell produces one figure. This makes it easy to re-run a single plot, debug layout issues, and keep notebook output clean.

### ✅ Good: separate cells for each figure

```python
# Cell: Flood depth map for RP100
fig, ax = plt.subplots(1, 1, figsize=(10, 8), subplot_kw={'projection': ccrs.PlateCarree()})
flood_maps['RP100'].plot(ax=ax, cmap='YlOrRd', vmin=0, vmax=5)
ax.set_title(f"Flood Depth Map — RP100 | {BASIN_ID}")
ax.coastlines()
plt.savefig(VIZ_OUTPUT / "flood_depth_RP100.png")
plt.show()
```

```python
# Cell: Exceedance probability curve
fig, ax = plt.subplots(figsize=(8, 5))
ax.semilogy(return_periods, mean_depths, 'o-', color='steelblue')
ax.set_xlabel("Return Period (years)")
ax.set_ylabel("Mean Flood Depth (m)")
ax.set_title(f"Exceedance Curve — {BASIN_ID}")
plt.savefig(VIZ_OUTPUT / "exceedance_curve.png")
plt.show()
```

### ❌ Bad: multiple plots in one cell

```python
# Two plt.show() in one cell — hard to debug, messy output
plt.figure()
plt.plot(x, y)
plt.show()

plt.figure()
plt.bar(categories, values)
plt.show()
```

## Use Cartopy for Map Projections

For geospatial plots, always use cartopy for proper map projections. Plain matplotlib imshow doesn't handle CRS correctly.

### ✅ Good: cartopy with coastlines and features

```python
fig, ax = plt.subplots(
    figsize=(10, 8),
    subplot_kw={'projection': ccrs.PlateCarree()}
)

# Plot flood depth
im = flood_depth.plot(
    ax=ax,
    transform=ccrs.PlateCarree(),
    cmap='YlOrRd',
    vmin=0, vmax=5,
    add_colorbar=True,
    cbar_kwargs={'label': 'Flood Depth (m)', 'shrink': 0.8},
)

# Add context
ax.coastlines(linewidth=0.5)
ax.add_feature(cfeature.BORDERS, linewidth=0.3, linestyle='--')
ax.add_feature(cfeature.RIVERS, linewidth=0.3, color='steelblue')

# Add boundary overlay
boundary_gdf.boundary.plot(ax=ax, color='black', linewidth=1.5, transform=ccrs.PlateCarree())

ax.set_title(f"Flood Depth — RP{rp} | {BASIN_ID}", fontweight='bold')
ax.set_extent([lon_min, lon_max, lat_min, lat_max])
```

### ❌ Bad: plain imshow without projection

```python
plt.imshow(flood_depth.values)  # No CRS, no coastlines, flipped y-axis
plt.colorbar()
```

## Use Gridspec for Multi-Panel Layouts

For comparison plots (multiple return periods, before/after), use gridspec for precise layout control.

### ✅ Good: 3x2 grid of flood depth maps

```python
fig = plt.figure(figsize=(15, 12))
gs = gridspec.GridSpec(3, 2, figure=fig, hspace=0.3, wspace=0.2)

return_periods_to_plot = [10, 20, 50, 100, 200, 500]

for idx, rp in enumerate(return_periods_to_plot):
    row, col = divmod(idx, 2)
    ax = fig.add_subplot(gs[row, col], projection=ccrs.PlateCarree())

    flood_maps[f'RP{rp}'].plot(
        ax=ax, cmap='YlOrRd', vmin=0, vmax=5,
        add_colorbar=(col == 1),  # Colorbar only on right column
    )
    ax.coastlines(linewidth=0.5)
    boundary_gdf.boundary.plot(ax=ax, color='black', linewidth=1)

    # Stats box
    mean_d = float(flood_maps[f'RP{rp}'].mean())
    max_d = float(flood_maps[f'RP{rp}'].max())
    ax.text(0.02, 0.98, f"Mean: {mean_d:.2f}m\nMax: {max_d:.1f}m",
            transform=ax.transAxes, va='top', fontsize=8,
            bbox=dict(boxstyle='round', facecolor='white', alpha=0.8))

    ax.set_title(f"RP {rp} years", fontweight='bold')

fig.suptitle(f"Flood Depth Maps by Return Period — {BASIN_ID}", fontsize=14, fontweight='bold')
plt.savefig(VIZ_OUTPUT / "02_flood_depth_maps_by_rp.png")
plt.show()
```

## Use Folium for Interactive Maps

For stakeholder-facing outputs and exploration, create interactive HTML maps with folium. These can be shared without requiring Python.

### ✅ Good: interactive map with colored markers

```python
import folium
from folium.plugins import MarkerCluster

# Create base map centered on study area
m = folium.Map(
    location=[center_lat, center_lon],
    zoom_start=10,
    tiles='OpenStreetMap',
)

# Add flood depth markers with color coding
for _, row in stations_gdf.iterrows():
    color = 'red' if row['depth'] > 2.0 else 'orange' if row['depth'] > 0.5 else 'green'
    folium.CircleMarker(
        location=[row.geometry.y, row.geometry.x],
        radius=6,
        color=color,
        fill=True,
        fill_opacity=0.7,
        popup=f"Station: {row['name']}<br>Depth: {row['depth']:.2f}m",
    ).add_to(m)

# Save to HTML
html_path = VIZ_OUTPUT / "interactive_flood_map.html"
m.save(str(html_path))
print(f"✓ Interactive map saved: {html_path.name}")
```

## Always Save Figures to Disk

Inline display is for exploration. For any figure you want to share or include in a report, save it to disk at 300 DPI.

### ✅ Good: save then show

```python
fig_path = VIZ_OUTPUT / "exceedance_curve.png"
plt.savefig(fig_path, dpi=300, bbox_inches='tight', facecolor='white')
print(f"✓ Saved: {fig_path.name}")
plt.show()
```

### ❌ Bad: only showing inline

```python
plt.show()
# Figure is gone — can't include in report without re-running
```

### Create the output directory early

```python
VIZ_OUTPUT = OUTPUT_DIR / "figures"
VIZ_OUTPUT.mkdir(parents=True, exist_ok=True)
```

## Use Domain-Appropriate Colormaps

Choose colormaps that communicate meaning:

| Data type | Colormap | Rationale |
|-----------|----------|-----------|
| Flood depth | `YlOrRd` | Yellow=shallow, Red=deep — intuitive danger scale |
| Elevation | `terrain` | Green=low, Brown=high — familiar topographic convention |
| Temperature anomaly | `RdBu_r` | Blue=cold, Red=hot — diverging around zero |
| Probability | `viridis` | Perceptually uniform, readable in print |
| Binary (flood/no-flood) | `Blues` with 2 levels | Simple presence/absence |

### ❌ Bad: rainbow colormap

```python
plt.imshow(data, cmap='jet')  # Perceptually non-uniform, misleading
```

### ✅ Good: domain-appropriate

```python
plt.imshow(flood_depth, cmap='YlOrRd', vmin=0, vmax=5)
```

## Add Contextual Information to Plots

Plots should be self-explanatory. Add stats boxes, titles with metadata, and boundary overlays.

```python
# Stats annotation box
stats_text = (
    f"Basin: {BASIN_ID}\n"
    f"Mean depth: {mean_depth:.2f} m\n"
    f"Max depth: {max_depth:.1f} m\n"
    f"Flooded area: {flooded_area_km2:.0f} km²"
)
ax.text(0.02, 0.98, stats_text,
        transform=ax.transAxes, va='top', fontsize=9,
        fontfamily='monospace',
        bbox=dict(boxstyle='round', facecolor='lightyellow', alpha=0.9, edgecolor='gray'))
```

## Separate Exploration Plots from Final Figures

Use a simple convention: exploration plots are inline-only; final figures are saved.

```python
# Exploration (quick check, not saved)
flood_maps['RP100'].plot(figsize=(6, 4))
plt.title("Quick check: RP100 coverage")
plt.show()
```

```python
# Final figure (saved, publication quality)
fig, ax = plt.subplots(figsize=(10, 8), subplot_kw={'projection': ccrs.PlateCarree()})
# ... full formatting ...
plt.savefig(VIZ_OUTPUT / "figure_01_flood_depth.png", dpi=300)
plt.show()
```

## Checklist

| Aspect | Target |
|--------|--------|
| rcParams setup | Set font sizes, DPI, line widths globally at start of viz section |
| One plot per cell | Each visualization in its own cell |
| Cartopy projections | `ccrs.PlateCarree()` with coastlines, borders |
| Gridspec multi-panel | Multi-panel grids with per-panel stats |
| Folium interactive | Circle markers colored by value, HTML output for stakeholders |
| Save to disk | All figures saved to `figures/` at 300 DPI |
| Domain colormaps | `YlOrRd` for depth, `viridis` for continuous, never `jet` |
| Stats annotations | Mean/max/area boxes on each panel |
| Output directory | Dedicated `figures/` directory created early |
| Exploration vs final | Quick inline checks separate from publication figures |
