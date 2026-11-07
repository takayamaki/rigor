# Answer `params.require` and `params.expect` as the gradual `params[...]` read type, and accept the lost protection on chains through them

- **PR:** none
- **Issue:** [#2947](https://github.com/rigortype/rigor/issues/2947)
- **Class:** open-choice
- **Precedent:** [#2445](https://github.com/rigortype/rigor/pull/2445)
- **Decided:** 2026-11-07

## Background

rigor-actionpack answered `params.require` and `params.expect` as the bare nominal `ActionController::Parameters`. In Rails both return the value itself: a scalar, an Array, or `false` (`value.present? || value == false`). With RBS for `Parameters` loaded (the actionpack 8.1 entry of ruby/gem_rbs_collection), `params.expect(:sort).to_sym` reported `call.undefined-method` on correct code. The issue suggested a gradual answer, as #2430 gave `params[...]`, without saying which one. The choice costs protection on correct-code chains, and nobody had weighed that.

## Options and trade-offs

- (a) Keep the nominal. It keeps every report and the coverage, and it keeps the false positive.
- (b) `Dynamic[Parameters]`, the type the carrier rules give `permit`. It was built first and sent back in review. `RbsDispatch#facet_descriptor` projects that facet when RBS is loaded, so `.dup`, `.clone`, `.itself`, `.freeze`, `.then` and `.tap`, and a declared `reject`, answer the bare nominal. `params.expect(:x).dup.to_sym` and `params.require(:ids).reject(&:blank?).map(&:to_i)` still reported, and `params.require(:flag).itself ? :on : :off` folded although `require` can return `false`.
- (c) `Dynamic[Parameters | nil]`, the type `params[...]` answers. The engine does not project a `Dynamic` over a union, so all of those shapes are silent, and the plugin's carrier rules keep `.permit` and `.slice` resolving. The facet is nilable only for this reason: the methods raise rather than return nil. It costs what (b) costs on coverage and on the true positive below.
- (d) Decline the rule for both methods, so the RBS-declared `untyped` answers, which the issue also offered. Not built: the answer would carry no `Parameters` facet, so the carrier rules could not fire and `params.require(:user).permit(:name)` would stop resolving.

## Decision

(c). False positives outrank protection (AGENTS.md), and #2445 made the same trade for `params[...]`, whose entry the maintainer ratified.

These consequences are accepted and not fixed:

- `coverage --protection` falls on chains through `require` and `expect`, because the sites after them have a `Dynamic` receiver. On a small controller fixture (seven lines of chains, with and without RBS for `Parameters`) the count went from 16 / 16 protected before #2947 to 8 / 16. A review fixture went from 26 protected to 13. (b) measured the same 8 / 16 on the small fixture.
- `params.require(:u).permit(:name).no_such_method` no longer reports. `params.permit(:name).no_such_method` still does, because `permit` on the `params` receiver keeps the nominal. `permit` on the carrier keeps the answer #2445 pinned.
- A kept-nil arm of a `.nil?` test now reports, though the arm is dead because both methods raise instead of returning nil. `x = params.require(:a); y = x.nil? ? nil : "s"; y.upcase` draws `call.possible-nil-receiver`. The `nil` in the facet is not the cause: `.nil?` narrows any `Dynamic` to `nil`, and a declared `-> untyped` value reports the same way.
- `rigor sig-gen` skips `def c = params.require(:u).permit(:a)` as `sig.skipped.untyped-return`, where it used to write `-> ActionController::Parameters`. It also no longer writes the wrong `-> Parameters` for `def a = params.require(:x)`.
- Outside a controller scope the carrier rules do not apply, so `ActionController::Parameters.new(a: 1).require(:x).permit(:a)` answers `Dynamic[top]` and a typo after it no longer reports.

The fit with the precedent is close but not exact: #2445 changed one read, where this also moves two entries of the strong-parameters chain.

## To undo

Remove `SCALAR_CAPABLE_CHAIN_METHODS` and its two `next params_read_type` lines in `plugins/rigor-actionpack/lib/rigor/plugin/actionpack.rb`. Both methods then answer what they answered before, the false positive returns, and the #2947 examples in `spec/integration/plugins/actionpack_plugin_spec.rb` fail. The answer depends on the engine not projecting `Dynamic[Union]`: [#2448](https://github.com/rigortype/rigor/issues/2448), which proposes that projection, would have to keep these examples passing, or `.dup.to_sym` reports again.
