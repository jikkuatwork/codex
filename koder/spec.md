# Spec

- This is the repo of OpenAI Codex CLI
- I wonder whether it can be enhanced to be an MCP client
- docs for mcp is available in `./docs/mcp/`

## Requirement

Analyze whether it would be possible and do a report in analysis.md; with the
following:

- effort
- feasibility
- ability to extend with minimial code change
- suggest strategy (say, import the package & create a new one with mcp
  capability? or extend this?)

Note

- I would prefer if it can be done such that future changes can be pulled in
  easily & merged

## Loud Thought

- Looks like there is an mcp sdk
- I wonder whether it can be used to easily implement a mcp client inside codex

## Status

The analysis is complete, but can you extend it?

Can we extend the analysis with a section: Phased Development? I would like to
split the implementation into several phases optimising for small early wins. I
want to test something simple super early. Say: ability to add a simple MCP,
call, execute it and see the results. Feel free to make it lack as many features
as possible, but at every phase, we will have something to see & test. Also, I
would prioritise visual, features as compared to deeper architectural ones.
