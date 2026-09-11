(ns laundrycoord.store
  "SSoT for the ISCO-08 8157 laundry machine operators plant
  scheduling/logistics coordination actor (itonami actor pattern,
  ADR-2607121000 / CLAUDE.md Actors section; README's 'Robotics
  premise' — a plant scheduling/logistics coordination robot performs
  crew scheduling, production-run/inventory/progress-record logging
  and detergent/linen-stock supply-order coordination for an
  industrial laundry crew under this advisor/governor pair, which
  never dispatches hardware itself, never operates laundry-machine
  equipment itself, and never finalizes a machine-operation-execution
  decision or a plant-safety-clearance decision, and never overrides a
  plant safety officer's judgment — those remain the plant safety
  officer's exclusive judgment). Modeled closely on
  cloud-itonami-isco-8152's millcoord.store.

  Domain:

    launderer — a registered laundry machine operator crew member
                (:launderer-id, :name). ('launderer' here names the
                closed-allowlist crew-record entity distinctly from
                'operator' the deploying/certified business entity
                used elsewhere in this repo's docs — see the Robotics
                premise in README.md — and distinctly from the ISCO
                occupation title 'Laundry Machine Operators' itself.)
    plant      — a registered industrial laundry plant/line
                 {:plant-id :name :max-supply-cost number}.
                 `:max-supply-cost` is an informational registered
                 ceiling used only to decide whether a
                 `:coordinate-supply-order` proposal escalates to
                 human sign-off (the governor never blocks a
                 within-threshold order outright; it only decides
                 commit vs. escalate).
    record     — a committed operating record (a logged production-
                 run/inventory/progress entry, a scheduled crew/shift
                 operation, a flagged safety concern, or a coordinated
                 detergent/linen-stock supply order) — written ONLY
                 via commit-record!. This actor coordinates plant
                 scheduling/logistics ONLY — a `record` is a
                 coordination artifact, never a
                 machine-operation-execution act, never a
                 plant-safety-clearance decision, and never a plant
                 safety officer's-judgment override.
    ledger     — append-only audit trail, commit or hold.")

(defprotocol Store
  (launderer [s launderer-id])
  (plant [s plant-id])
  (records-of [s launderer-id])
  (ledger [s])
  (register-launderer! [s launderer])
  (register-plant! [s plant])
  (commit-record! [s record])
  (append-ledger! [s fact]))

(defrecord MemStore [a]
  Store
  (launderer [_ launderer-id] (get-in @a [:launderers launderer-id]))
  (plant [_ plant-id] (get-in @a [:plants plant-id]))
  (records-of [_ launderer-id] (filter #(= launderer-id (:launderer-id %)) (:records @a)))
  (ledger [_] (:ledger @a))
  (register-launderer! [s l]
    (swap! a assoc-in [:launderers (:launderer-id l)] l) s)
  (register-plant! [s p]
    (swap! a assoc-in [:plants (:plant-id p)] p) s)
  (commit-record! [s record]
    (swap! a update :records (fnil conj []) record) s)
  (append-ledger! [s fact]
    (swap! a update :ledger (fnil conj []) fact) s))

(defn mem-store
  ([] (mem-store {}))
  ([seed] (->MemStore (atom (merge {:launderers {} :plants {} :records [] :ledger []}
                                    seed)))))
