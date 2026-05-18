# Qt / GUI testing

Testing Qt-based GUIs from a Catch2 test binary lets GUI tests share the
same fixtures, factories, and reporters as the rest of the suite. `QSignalSpy`,
`QAbstractItemModelTester`, `QTest::mouseClick`, `QTest::keyClicks`,
`QTest::qWait`, `QTest::qWaitFor`, and `QTest::qWaitForWindowExposed` are
plain library code in `Qt6::Test` and work under any `QCoreApplication`.

## The test binary owns the QApplication

Qt allows exactly one `QCoreApplication`-derived instance per process, and
reconstructing it is undefined behavior. A Qt test binary therefore
constructs the application once in `main`, before Catch2's session:

```cpp
#include <catch2/catch_session.hpp>

#include <QApplication>

int main(int argc, char* argv[])
{
  QApplication application{argc, argv};
  return Catch::Session().run(argc, argv);
}
```

Pick the application class that matches what the tests actually exercise:

| Tests touch                | Construct           |
|----------------------------|---------------------|
| `QWidget` (any subclass)   | `QApplication`      |
| QML / Qt Quick only        | `QGuiApplication`   |
| Pure non-GUI (signals, IO) | `QCoreApplication`  |

Mixing widgets into a `QGuiApplication` crashes at runtime. When unsure,
`QApplication` is the safe choice for a GUI test binary.

Forward `argc`/`argv` to both: Qt consumes its own flags during construction,
then Catch2 parses what remains. Exact flag handling can vary by Qt version.

## CMake for a GUI test

A GUI test target differs from a regular test in three ways: it links
`Catch2::Catch2` (no main) instead of `Catch2::Catch2WithMain`, it links
`Qt6::Test` for `QSignalSpy` and `QTest::*`, and it sets
`QT_QPA_PLATFORM=offscreen` so CI runs without a display.

```cmake
add_executable(settings_dialog_test
  src/main.cpp
  src/test_settings_dialog.cpp
)

target_link_libraries(settings_dialog_test PRIVATE
  Catch2::Catch2
  Qt6::Widgets
  Qt6::Test
  settings_dialog
)

catch_discover_tests(settings_dialog_test
  PROPERTIES ENVIRONMENT "QT_QPA_PLATFORM=offscreen"
)
```

## Headless CI: the offscreen platform

Set `QT_QPA_PLATFORM=offscreen` for GUI test targets. Every
widget gets a real event dispatcher, real geometry, and real signal/slot
delivery without opening a window. It covers the bulk of widget tests:
geometry, layout, model/view, signal flow, and simulated input that targets a
specific widget.

Use `xvfb-run -a ./<binary>` when offscreen drops events you need to test:

- Real keyboard-focus paths -- `QLineEdit::editingFinished` may not fire
  from simulated input under offscreen because the widget never gets
  activation. Calling `widget.setFocus()` and
  `QApplication::setActiveWindow(&widget)` before sending input usually
  works around this without xvfb.
- Tooltips, hover, mouse-tracking -- simulated hover events do not
  propagate the same way under offscreen.
- Drag-and-drop across windows, system tray, native dialogs -- anything
  that round-trips through a real compositor.
- OpenGL / Qt Quick scenegraph -- pair with `QT_QUICK_BACKEND=software`.

Avoid the `minimal` platform plugin for widget tests; it strips event
delivery to widgets, so input-driven tests look like they pass while doing
nothing.

## Showing widgets and waiting for them

Simulated input goes to whichever widget owns the geometry under the event
coordinates. If the widget has not been shown and laid out, the input is
dropped. Always show, then wait:

```cpp
QPushButton button{"OK"};
button.show();
REQUIRE(QTest::qWaitForWindowExposed(&button));
```

The call is cheap under offscreen and pumps the events the widget needs
before input; always make it.

## Pumping the event loop

Without QTest's runner, the event loop has to be advanced explicitly. Pick
the call that matches the wait:

| Need                                                | Use                                                  |
|-----------------------------------------------------|------------------------------------------------------|
| Drain pending events synchronously, no waiting      | `QCoreApplication::processEvents()`                  |
| Wait a fixed wall-clock duration with events pumped | `QTest::qWait(50)` (milliseconds)                    |
| Wait for a predicate to become true (poll + timeout)| `QTest::qWaitFor([&]{ return cond; }, 1000)`         |
| Wait for a window to be ready for input             | `QTest::qWaitForWindowExposed(&widget)`              |
| Wait for one specific signal                        | `QSignalSpy spy{...}; spy.wait(timeout)`             |

`qWaitFor` is the standalone-safe replacement for `QTRY_VERIFY`. Prefer
waiting for an observable condition over waiting a fixed duration -- fixed
waits are flaky on loaded CI and slow tests down for no benefit when the
work finishes faster.

## Simulated input

`QTest::mouseClick`, `QTest::keyClick`, `QTest::keyClicks`,
`QTest::mouseMove`, and `QTest::sendEvent` all work outside `QTEST_MAIN`.

```cpp
QPushButton button{"Submit"};
button.show();
REQUIRE(QTest::qWaitForWindowExposed(&button));

QSignalSpy clicked_spy{&button, &QPushButton::clicked};
QTest::mouseClick(&button, Qt::LeftButton);
CHECK(clicked_spy.count() == 1);
```

For typing into an editor widget, give it focus first; offscreen does not
focus a widget on its own:

```cpp
QLineEdit edit;
edit.show();
REQUIRE(QTest::qWaitForWindowExposed(&edit));
edit.setFocus();

QTest::keyClicks(&edit, "hello");
CHECK(edit.text() == "hello");
```

Widgets inside item views (`QTreeView`, `QTableView`, `QListView`) receive
input on the `viewport()`, not on the items themselves. Compute the click
point from `visualRect(index).center()`:

```cpp
auto index = model.index(2, 0);
auto point = view.visualRect(index).center();
QTest::mouseClick(view.viewport(), Qt::LeftButton, Qt::NoModifier, point);
```

## Observing signals: QSignalSpy

`QSignalSpy` is a small `QObject` that connects to a signal and records each
emission as a `QList<QVariant>`. Use the pointer-to-member form; the macro
`SIGNAL(...)` form resolves at runtime and silently no-ops on typos or
overload mismatches.

```cpp
QSignalSpy data_changed_spy{&model, &QAbstractItemModel::dataChanged};

model.setData(model.index(0, 0), QStringLiteral("renamed"));

REQUIRE(data_changed_spy.count() == 1);
auto args      = data_changed_spy.takeFirst();
auto top_left  = args.at(0).value<QModelIndex>();
auto roles     = args.at(2).value<QList<int>>();
CHECK(roles.contains(Qt::DisplayRole));
```

Overloaded signals need `qOverload`. The example below uses an application
widget with two `submitted` overloads:

```cpp
QSignalSpy spy{&form, qOverload<int>(&login_form::submitted)};
```

A spy constructed against a non-existent or mistyped signal records nothing
and never fails, so sanity-check `spy.isValid()` once during bring-up.

For cross-thread or queued connections (`QTimer::singleShot`, queued
`QMetaObject::invokeMethod`, `deleteLater`), use `spy.wait(timeout)`:

```cpp
QSignalSpy finished_spy{&worker, &file_loader::finished};
worker.start();
REQUIRE(finished_spy.wait(5000));
CHECK(finished_spy.count() == 1);
```

Joining a `QThread` with `wait()` does not pump the test thread's event
loop, so a queued completion signal can be dropped if the join happens
before the spy. Pattern: subscribe with `QSignalSpy`, trigger work,
`spy.wait`, then `thread.quit(); thread.wait()`.

## Object lifetime

Catch2 sections share the `QApplication`, so `QObject` leaks accumulate
across the whole binary's lifetime. Use a stack-allocated parent with
heap-allocated children parented to it, or `std::unique_ptr<QWidget>`:

```cpp
TEST_CASE("login_form emits submitted when accept button is clicked",
          "[gui][login_form]")
{
  QWidget container;
  auto* form = new login_form{&container};
}
```

- Do not give a stack-allocated child a stack-allocated parent. The parent
  will try to delete the child during destruction. Pick one ownership style
  per object.
- `deleteLater` runs only when the event loop turns. If a test depends on a
  `deleteLater`'d object being gone, pump the loop first.

## Model/View testing

For custom `QAbstractItemModel` subclasses, layer two checks:

1. `QAbstractItemModelTester` as a passive invariant checker, constructed
   *before* the mutations so its connected slots see every change.
2. Behavior tests on top, asserting against expected role data, signal
   payloads, and structural changes.

```cpp
TEST_CASE("file_list_model inserts a row when a file is added",
          "[gui][file_list_model]")
{
  file_list_model model;
  QAbstractItemModelTester tester{
      &model, QAbstractItemModelTester::FailureReportingMode::Fatal};

  QSignalSpy rows_inserted_spy{&model, &QAbstractItemModel::rowsInserted};

  model.add_file("/tmp/example.txt");

  CHECK(model.rowCount() == 1);
  CHECK(rows_inserted_spy.count() == 1);
}
```

Use `FailureReportingMode::Fatal` (aborts on invariant violation) or
`Warning` (logs to `qt.modeltest`). The default `QtTest` mode reports
through QTest's logger, which Catch2 will not surface.

Pair `QAbstractItemModelTester` with explicit behavior tests that assert
insertion, removal, move, `dataChanged` payloads, role coverage, and
`headerData`.

## High-DPI

Qt6 enables high-DPI scaling by default. Hard-coded pixel coordinates break
across platforms. Use `widget.rect().center()` and similar widget-relative
geometry instead.

## See also

- `catch2-conventions.md` -- Catch2 binary and assertion conventions.
- `condition-based-waiting.md` -- framework-agnostic predicate waits.
- `test-patterns.md` -- structuring GUI scenarios with `TEST_CASE` and
  `SECTION`.
