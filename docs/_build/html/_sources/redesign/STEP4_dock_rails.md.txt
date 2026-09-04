# Step 4.1 — Collapsible rails (patch)

§2.6 is in and correct: the colorbar shares the image's top and bottom edge, its labels are
readable, and the axes sit on the data's edge. Two cosmetic residuals to sweep while you are
in this file — both are tick-label placement, not geometry:

- bottom-left corner, `104.80` and `68.54` collide. Drop the **first** x tick label when
  `data.x - (width of the y tick label) < 8 px`; the corner value is readable from the y
  axis anyway. Cheapest fix: skip `i == 0` for the x row when the y labels are visible.
- `Galactic Longitude` touches the tick row. The title sits at `bottomMargin / 2 - 2`, which
  was tuned when `bottomMargin` was the full 58 px from the window edge; now it is
  `data.y`, which is larger. Anchor it to the window instead: `gutter.bottom / 2 - 2`.

On BUNIT: fetch it if the header response already carries it (`Spectral axis: Radio Velocity
(m/s)` says the parsing exists, so the keyword is probably reachable). A brightness value
with no unit is ambiguous in a readout meant to be quoted — but a wrong unit is worse, so
keep the current behaviour as the fallback.

---

## What this step is

Both docks collapse to a **36 px rail — never 0, never hidden**. `VisivoTheme::kRailWidth`
already exists (36) alongside `kLeftDockWidth` (272) and `kRightDockWidth` (320), so the
numbers are in place; nothing here should introduce a new literal.

Defaults change with this step: **left open, right collapsed.** Threshold, blend mode and
LUT are touched every few seconds; the Inspector is only needed when starting a task.

Reference: `#12a` (default state), `#12b` (focus mode), §4.1 of `HANDOFF_cube_viewer.md`.

---

## 4.1.1 The rail widget

New file pair `src/gui/DockRail.{h,cpp}`. One widget, used for both sides — the right one
just gets extra items.

```cpp
/// The 36 px collapsed state of a dock: expand glyph, rotated panel name,
/// optional rotated clickable entries (the Inspector's tab names), and a count
/// of what is behind the rail so a closed panel can still report something.
class DockRail : public QWidget
{
    Q_OBJECT
public:
    explicit DockRail(const QString &title, const QString &expandGlyph, QWidget *parent = nullptr);

    /// Rotated, clickable entries under the title (right rail only).
    void setEntries(const QStringList &entries);
    void setActiveEntry(int index);
    /// Mono 9 px number at the bottom — products (left) or running tasks (right).
    void setCount(int count);

signals:
    void expandRequested();
    void entryClicked(int index);   ///< expands the dock AND selects that tab

protected:
    void paintEvent(QPaintEvent *) override;
    void mousePressEvent(QMouseEvent *) override;
    QSize sizeHint() const override { return { VisivoTheme::kRailWidth, 200 }; }

private:
    QString m_title;
    QString m_glyph;
    QStringList m_entries;
    int m_active{ -1 };
    int m_count{ 0 };
    QRect m_glyphRect;
    QList<QRect> m_entryRects;   ///< filled in paintEvent, hit-tested in mousePress
};
```

Painting, top to bottom — the layout is fixed, so paint it directly rather than fighting a
`QVBoxLayout` full of rotated widgets:

1. the expand glyph (`◧` left / `◨` right), `kPrimary()`, 13 px, in a 36 × 28 box at the top.
   Store its rect in `m_glyphRect`;
2. a 20 × 1 px `kOutlineVariant()` rule, centred;
3. the panel name, rotated 90°, Lato 10 px / 700, letter-spacing 0.14em,
   `kOnSurfaceVariant()`;
4. stretch;
5. the entries, rotated the same way, active one in `kPrimary()` / 600, the rest
   `kOnSurfaceVariant()`. Push each item's *unrotated* bounding rect into `m_entryRects`;
6. the count, JetBrains Mono 9 px, centred, 8 px from the bottom.

Rotation, per side (the glyph and the count stay upright):

```cpp
    painter.save();
    painter.translate(VisivoTheme::kRailWidth / 2, y);
    painter.rotate(90);                     // reads top-to-bottom
    painter.drawText(QRect(0, -9, textLen, 18), Qt::AlignCenter, text);
    painter.restore();
```

Letter-spacing needs a `QFont` with `setLetterSpacing(QFont::PercentageSpacing, 114)` — the
section headers in the left dock already do this, match them.

`mousePressEvent`: `m_glyphRect.contains(pos)` → `expandRequested()`; otherwise the first
`m_entryRects` hit → `entryClicked(i)`. Set `Qt::PointingHandCursor` on the whole widget and
a tooltip per entry (the rotated text is small; the tooltip is the accessible affordance).

---

## 4.1.2 Docks become two-state

The dock's width is driven by its child's min/max — both containers are pinned
(`setMinimumWidth == setMaximumWidth`), so collapsing means swapping the child, not resizing
the dock. Wrap each in a `QStackedWidget`: page 0 the existing content, page 1 the rail.

In `WorkspaceChrome::installSessionDock()`, after `container` is fully built:

```cpp
    m_sessionStack = new QStackedWidget(m_sessionDock);
    m_sessionRail = new DockRail(QObject::tr("SESSION DATA"), u"\u25E7"_s, m_sessionStack);
    m_sessionStack->addWidget(container);      // page 0
    m_sessionStack->addWidget(m_sessionRail);  // page 1
    m_sessionDock->setWidget(m_sessionStack);
    QObject::connect(m_sessionRail, &DockRail::expandRequested,
                     this, [this]() { setSessionCollapsed(false); });
```

and the same for the inspector (`u"\u25E8"_s`, entries `{Properties, Analysis, Provenance}`).

The pinned widths move from the content widgets to the stack, because the stack is now what
the dock measures:

```cpp
void WorkspaceChrome::applyDockWidth(QStackedWidget *stack, bool collapsed, int openWidth)
{
    if (!stack)
        return;
    const int w = collapsed ? VisivoTheme::kRailWidth : openWidth;
    stack->setCurrentIndex(collapsed ? 1 : 0);
    stack->setMinimumWidth(w);
    stack->setMaximumWidth(w);
}
```

Leave the `setMinimum/MaximumWidth` on `container` and on `m_inspector` as they are — they
are inside the stack and still correct for page 0.

Public API on `WorkspaceChrome`:

```cpp
    void setSessionCollapsed(bool collapsed);
    void setInspectorCollapsed(bool collapsed);
    bool sessionCollapsed() const { return m_sessionCollapsed; }
    bool inspectorCollapsed() const { return m_inspectorCollapsed; }
    /// Expand + select a tab. Called by the rail entries and by task launches.
    void revealInspectorTab(int index);

signals:
    void dockCollapseChanged();   ///< so the window can re-fit the 2-D panes
```

`setInspectorCollapsed()` also hides the custom title bar (see below) and, when expanding,
keeps whatever tab was last selected. **Never auto-collapse** — only the user's click,
focus mode, or the restored setting closes it.

Connect `dockCollapseChanged()` to `layout2dPane()`: collapsing a dock resizes the panes,
and the gutter fit is size-driven. The `QEvent::Resize` filter from §2.4 will usually fire
anyway, but not reliably when both docks change in the same event loop turn.

---

## 4.1.3 The expand/collapse affordance in the header

The docks currently use the default `QDockWidget` title bar, which has no room for a button
and cannot be styled to the spec. Give both a custom one — this is also what lets the header
disappear cleanly in the collapsed state:

```cpp
QWidget *WorkspaceChrome::makeDockTitleBar(const QString &title, const QString &glyph,
                                           const std::function<void()> &onToggle)
{
    auto *bar = new QWidget();
    bar->setFixedHeight(34);
    auto *h = new QHBoxLayout(bar);
    h->setContentsMargins(14, 0, 8, 0);
    auto *label = new QLabel(title, bar);
    label->setStyleSheet(QStringLiteral("color:%1; font-family:'Lato'; font-size:12px;")
                                 .arg(QLatin1StringView(VisivoTheme::kOnSurface())));
    h->addWidget(label);
    h->addStretch(1);
    auto *btn = new QToolButton(bar);
    btn->setText(glyph);
    btn->setCursor(Qt::PointingHandCursor);
    btn->setToolTip(QObject::tr("Collapse panel"));
    btn->setStyleSheet(QStringLiteral("QToolButton{background:transparent;border:none;"
                                      "color:%1;font-size:13px;}QToolButton:hover{color:%2;}")
                               .arg(QLatin1StringView(VisivoTheme::kOnSurfaceVariant()),
                                    QLatin1StringView(VisivoTheme::kPrimary())));
    QObject::connect(btn, &QToolButton::clicked, bar, onToggle);
    h->addWidget(btn);
    return bar;
}
```

`m_sessionDock->setTitleBarWidget(...)` / `m_inspectorDock->setTitleBarWidget(...)`, and in
the collapsed state `dock->titleBarWidget()->setVisible(false)` — a 34 px bar reading
"Session Data" above a 36 px rail would clip, and the rail already carries the name.

Note this replaces the drag handle, so keep `DockWidgetMovable` off the features of both
docks while collapsed, or Qt will let the user float a 36 px rail.

---

## 4.1.4 Defaults and auto-expand

At the end of `installInspectorDock()`:

```cpp
    setInspectorCollapsed(true);    // right starts as a rail (spec §4.1)
```

The Inspector auto-expands (`revealInspectorTab()`) on exactly three triggers, all of which
already exist as call sites in `vtkWindowCube`:

| Trigger | Tab |
|---|---|
| a rail entry is clicked | that one |
| an Analysis tool is launched (`InspectorPanel::addTool`'s action fires) | Analysis |
| a task completes and registers a product | Provenance |

`InspectorPanel` needs two accessors it does not have yet — add them next to
`propertiesContentHeight()`:

```cpp
    void showTab(int index);
    int currentTab() const;
```

Feed the rail counts from data that is already tracked: left = product count from
`ProductRegistry`, right = the running count the status-rail poll already computes for the
queue slot. Reuse that value; do not add a second poll.

---

## 4.2 Focus mode

Small enough to land in the same pass. `⌥⇧F` on macOS, `Ctrl+Shift+F` elsewhere, as
"Focus mode" in the View menu, checkable:

```cpp
    const bool on = /* action checked */;
    m_workspaceChrome->setSessionCollapsed(on);
    m_workspaceChrome->setInspectorCollapsed(on);
```

It is a toggle, not a mode: leaving it restores both docks to the state they had before,
so remember the pair before collapsing. Also thin the status rail to health + queue while
active (hide the cache and session slots) — they are the two that change.

Skip §4.3 (`QSettings` persistence) for now; it is the next patch and wants the pane
assignment from Step 3 in the same key block, `layout_v2/cube/…`.

---

## Acceptance

1. Left dock open at 272, right side a 36 px rail reading `INSPECTOR` vertically, with
   `Properties / Analysis / Provenance` below it and a `0` at the bottom.
2. Clicking `Analysis` on the rail expands the dock to 320 **and** selects that tab.
3. `◧` in the Session Data header collapses the left dock to a rail reading `SESSION DATA`
   with the product count; its `◧` restores it to exactly 272.
4. Neither dock can reach 0 px, and neither can be dragged out while collapsed.
5. Launching a tool from Analysis expands the Inspector by itself; nothing ever collapses it.
6. `⌥⇧F` collapses both to a ≈ 1348 px viewport and restores the previous pair.
7. With both collapsed, the 2-D pane re-fits: the image is still centred and its axes still
   sit on its edge, i.e. `layout2dPane()` ran.

---

## 4.5 The collapsed dock is 36 px, but the dock *area* is not

A collapsed rail leaves ~150 px of dead space between itself and the pane frame — the rail
paints at 36 px, but `QMainWindow` keeps the dock area much wider. Two things still report a
larger minimum, and both must go:

**1. The hidden title bar.** `setVisible(false)` does not remove a title-bar widget from
`QDockWidget`'s minimum-width calculation. 150 px is almost exactly "Session Data" + 14 px
margin + the glyph button, which is the tell. Swap the widget instead of hiding it:

```cpp
    // Collapsed: an empty 0-height bar. The rail carries the name and the glyph,
    // so there is nothing to show — and nothing to contribute a minimum width.
    if (collapsed) {
        auto *empty = new QWidget(dock);
        empty->setFixedHeight(0);
        dock->setTitleBarWidget(empty);      // deletes/replaces the previous bar
    } else {
        dock->setTitleBarWidget(makeDockTitleBar(...));
    }
```

Keep a `QPointer` to the real bar and re-use it rather than rebuilding on every toggle, or
rebuild it — either is fine, it is one widget.

**2. Page 0's pinned width.** `QStackedWidget::minimumSizeHint()` is the **maximum over all
pages**, not the current page, so `container`'s `setMinimumWidth(272)` keeps the stack's
minimum at 272 no matter what the rail asks for. Capping the stack with `setMaximumWidth(36)`
then produces a widget whose min exceeds its max, which Qt resolves in favour of the minimum.

Relax the page while it is not showing, in `applyDockWidth()`:

```cpp
    if (auto *page = stack->widget(0)) {
        page->setMinimumWidth(collapsed ? 0 : openWidth);
        page->setMaximumWidth(collapsed ? QWIDGETSIZE_MAX : openWidth);
    }
```

Check the result by measurement, not by eye — the same way the left dock is instrumented:

```cpp
    qDebug().noquote() << QStringLiteral("[dock] %1 collapsed=%2 stack=%3 dock=%4 area=%5")
            .arg(dock->objectName()).arg(collapsed)
            .arg(stack->width()).arg(dock->width())
            .arg(m_window->centralWidget()->x());   // where the panes actually start
```

Collapsed, all three must read 36 / 36 / 36 (plus the splitter handle). Anything larger means
a third widget in the chain is still pinned — `m_inspector` itself carries the same
`setMinimumWidth(kRightDockWidth)`, so it needs the same treatment as `container`.
