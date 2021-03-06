# Full Screen mode

We occasionally want to be able to temporarily full-screen (or "maximize") a
window without closing everything else on that screen, so let's implement that
today.

We'll use Ctrl+Alt+Enter as the key combination to toggle whether or not the
active window is maximized.

We've already done enough that this shouldn't be difficult: we'll have to grab
the key combo, and then send it a ConfigureWindow event to configure the window
to be the full size of the workspace that it's on. While we're at it, we should
remove the 2px border to really make sure it's full screen. Upon unmaximizing,
we should be able to just put the border back and call TileWindows to put things
back into their old position, since we didn't really touch anything else.

It seems to me that the obvious way to do this is to include a "maximizedWindow"
pointer on the workspace level, then in TileWindows we can just check if that's
non-nil and short-circuit out before tiling all the other windows. If it's nil,
we fall back on our old behaviour. We already have all the info we need (like
the screen size) in our Screen pointer struct in TileWindows().

Now, we'll have to start by updating our workspace type.

### "Workspace type"
```go
<<<Column type>>>
type Workspace struct{
	Screen *xinerama.ScreenInfo
	columns []Column

	maximizedWindow *xproto.Window

	mu *sync.Mutex
}
```

We'll add one simple if statement near the top of our latest tiling
implementation:

### "Tile Workspace Windows Implementation"
```go
if w.Screen == nil {
	return fmt.Errorf("Workspace not attached to a screen.")
}

if w.maximizedWindow != nil {
	<<<Resize *w.maximizedWindow and stack on top>>>
}
n := uint32(len(w.columns))
if n == 0 {
	return fmt.Errorf("No columns to tile")
}
var totalDeltas int
for _, c := range w.columns {
	totalDeltas += c.SizeDelta
}

size := uint32(int(w.Screen.Width)-totalDeltas) / n
var err error

// Keep track of the already incorporated deltas, to add to xstart
// for the column.TileWindow call
usedDeltas := 0
prevWin := activeWindow
for i, c := range w.columns {
	if err != nil {
		// Don't overwrite err if there's an error, but still
		// tile the rest of the columns instead of returning.
		c.TileColumn(uint32((i*int(size))+usedDeltas), uint32(int(size)+c.SizeDelta), uint32(w.Screen.Height))
	} else {
		err = c.TileColumn(uint32((i*int(size))+usedDeltas), uint32(int(size)+c.SizeDelta), uint32(w.Screen.Height))
	}
	usedDeltas += c.SizeDelta
}
if prevWin != nil {
	if err := xproto.WarpPointerChecked(xc, 0, *prevWin, 0, 0, 0, 0, 10, 10).Check(); err != nil {
		log.Print(err)
	}
}
return err
```

And then to Resize our window, we just make a ConfigureWindowChecked call with
the appropriate mask, similarly to what we do in TileColumns, but with the
screen width and height as the width and height, (0, 0) as the (X, Y) coordinates,
0 as the BorderWidth and we want to set the StackingMode to ensure we're on top
(normally we don't care since our windows don't overlap, but this is an exception.)

### "Resize *w.maximizedWindow and stack on top"
```go
return xproto.ConfigureWindowChecked(
	xc,
	*w.maximizedWindow,
	xproto.ConfigWindowX|
		xproto.ConfigWindowY|
		xproto.ConfigWindowWidth|
		xproto.ConfigWindowHeight|
		xproto.ConfigWindowBorderWidth|
		xproto.ConfigWindowStackMode,
	[]uint32{
		0,
		0,
		uint32(w.Screen.Width),
		uint32(w.Screen.Height),
		0,
		xproto.StackModeAbove,
	},
).Check()
```

Now, we need to grab our hotkey:

### "Grabbed Key List" +=
```go
{
	sym:       keysym.XK_Return,
	modifiers: xproto.ModMaskControl | xproto.ModMask1,
},
```

Add a key handler for it:

### "Keystroke Detail Switch" +=
```go
case keysym.XK_Return:
	<<<Handle Enter key>>>
```

And then do the handling.

Luckily, we now have a workspace.IsActive() method to determine which workspace
is active, so handling it should be fairly easy: if the active workspace has a
maximized window, unset it, put the 2px border back, and call TileWindows(). If
it doesn't, then set the maximizedWindow pointer to activeWindow and call
TileWindows() (the ConfigureWindow call from TileWindows will take care of the
border.)

### "Handle Enter key"
```go
switch key.State {
case xproto.ModMaskControl | xproto.ModMask1:
	for _, w := range workspaces {
		go func(w *Workspace) {
			if w.IsActive() {
				if w.maximizedWindow == nil {
					w.maximizedWindow = activeWindow
				} else {
					if err := xproto.ConfigureWindowChecked(
						xc,
						*w.maximizedWindow,
						xproto.ConfigWindowBorderWidth,
						[]uint32{2},
					).Check(); err != nil {
						log.Print(err)
					}
					w.maximizedWindow = nil
				}
				w.TileWindows()
			}
		}(w)
	}
}
return nil
```

This mostly works, except our pointer gets out of sync if we close a full-screen
window, resulting in a hole where our window used to be, and the next window
we create being maximized (but not having focus!). We should probably maintain
the pointer in the DestroyEvent handler.

DestroyEventNotifier always calls workspace.RemoveWindow, which is probably
an even better place to manage removing the pointer.

### "RemoveWindow implementation"
```go
wp.mu.Lock()
defer wp.mu.Unlock()

for colnum, column := range wp.columns {
	idx := -1
	for i, candwin := range column.Windows {
		if w == candwin.Window {
			idx = i
			break
		}
	}
	if idx != -1 {
		// Found the window at at idx, so delete it and return.
		// (I wish Go made it easier to delete from a slice.)
		wp.columns[colnum].Windows = append(column.Windows[0:idx], column.Windows[idx+1:]...)
		if wp.maximizedWindow != nil && w == *wp.maximizedWindow {
			wp.maximizedWindow = nil
		}
		return nil
	}	
}
return fmt.Errorf("Window not managed by workspace")
```


We should also update our go:generate directive to include this file:
### "Go generate directive"
```go
//go:generate lmt src/Initialize.md src/WindowManaging.md src/Keyboard.md src/MovingWindows.md src/ResizingWindows.md src/ColumnManagement.md src/OverrideRedirect.md src/GoGenerate.md src/Fullscreen.md
```

And now we're done. 