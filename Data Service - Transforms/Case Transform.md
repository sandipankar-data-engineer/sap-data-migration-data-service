# SAP Data Services -- Case Transform

## Purpose

The **Case** transform routes input rows to one or more output flows
according to Boolean conditions. Each case has a label and an
expression; rows satisfying an expression are sent to the corresponding
output.

## Configuration

-   **Case label:** Names an output flow (for example, `DEPT_10`).
-   **Expression:** A Boolean condition that determines whether a row is
    routed to that case (for example, `EMP.DEPTNO = 10`).
-   **Default output:** Optional output for rows that do not match any
    defined case.

## Important options

  |Option|Behaviour|
  |---|---|
  |**Produce default output**|Creates a default flow for rows that match none of the defined case conditions.|
  |**Row can be TRUE for one case only**|When selected, a row is routed to at most one case. When cleared, a row can be routed to every case whose condition evaluates to TRUE.|
  |**Preserve expression order**|Preserves the configured order of case-expression evaluation. This matters when expression order affects which case is selected; do not rely on the displayed order if this option is cleared.|
  -----------------------------------------------------------------------

## Example

For an employee dataset, define these cases:

  Output label   Expression
  -------------- -------------------
  `DEPT_10`      `EMP.DEPTNO = 10`
  `DEPT_20`      `EMP.DEPTNO = 20`
  `DEPT_30`      `EMP.DEPTNO = 30`

A row with `DEPTNO = 20` is routed to `DEPT_20`. A row that matches none
of these conditions goes to the default output, if enabled; otherwise it
is not sent to a case output.

If overlapping conditions are possible, clear **Row can be TRUE for one
case only** to allow a row to appear in multiple case outputs. Use
**Preserve expression order** when the configured evaluation order must
be retained.

## Input / Output

**1 input → multiple output flows (1:N).**
