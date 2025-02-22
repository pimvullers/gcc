;; Predicate description for RISC-V target.
;; Copyright (C) 2011-2018 Free Software Foundation, Inc.
;; Contributed by Andrew Waterman (andrew@sifive.com).
;; Based on MIPS target for GNU compiler.
;;
;; PULP family support contributed by Eric Flamand (eflamand@iis.ee.ethz.ch) at ETH-Zurich
;; and Greenwaves Technologies (eric.flamand@greenwaves-technologies.com)
;;
;; This file is part of GCC.
;;
;; GCC is free software; you can redistribute it and/or modify
;; it under the terms of the GNU General Public License as published by
;; the Free Software Foundation; either version 3, or (at your option)
;; any later version.
;;
;; GCC is distributed in the hope that it will be useful,
;; but WITHOUT ANY WARRANTY; without even the implied warranty of
;; MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
;; GNU General Public License for more details.
;;
;; You should have received a copy of the GNU General Public License
;; along with GCC; see the file COPYING3.  If not see
;; <http://www.gnu.org/licenses/>.

;; Return nonzero if OP is a LC register.
(define_predicate "lc_register_operand"
  (and (match_code "reg")
       (match_test "REGNO (op) == REG_LC0 || REGNO (op) == REG_LC1")))

;; Return nonzero if OP is a LE register.
(define_predicate "le_register_operand"
  (and (match_code "reg")
       (match_test "REGNO (op) == REG_LE0 || REGNO (op) == REG_LE1")))

;; Return nonzero if OP is a LS register.
(define_predicate "ls_register_operand"
  (and (match_code "reg")
       (match_test "REGNO (op) == REG_LS0 || REGNO (op) == REG_LS1")))

(define_predicate "permute_sel_operand"
  (ior (match_code "reg")
       (match_code "const_vector")))

(define_predicate "const_arith_operand"
  (and (match_code "const_int")
       (match_test "SMALL_OPERAND (INTVAL (op))")))

(define_predicate "arith_operand"
  (ior (match_operand 0 "const_arith_operand")
       (match_operand 0 "register_operand")))

(define_predicate "const_csr_operand"
  (and (match_code "const_int")
       (match_test "IN_RANGE (INTVAL (op), 0, 31)")))

(define_predicate "csr_operand"
  (ior (match_operand 0 "const_csr_operand")
       (match_operand 0 "register_operand")))

(define_predicate "sle_operand"
  (and (match_code "const_int")
       (match_test "SMALL_OPERAND (INTVAL (op) + 1)")))

(define_predicate "sleu_operand"
  (and (match_operand 0 "sle_operand")
       (match_test "INTVAL (op) + 1 != 0")))

(define_predicate "const_0_operand"
  (and (match_code "const_int,const_wide_int,const_double,const_vector")
       (match_test "op == CONST0_RTX (GET_MODE (op))")))

(define_predicate "reg_or_0_operand"
  (ior (match_operand 0 "const_0_operand")
       (match_operand 0 "register_operand")))

(define_predicate "reg_or_imm5_operand"
  (ior (match_operand 0 "register_operand")
       (and (match_code "const_int")
            (ior (match_operand 0 "const_0_operand")
                 (match_test "((Pulp_Cpu>=PULP_V2) && (INTVAL(op)>=-16) && (INTVAL(op)<=15))")
            )
       )
  )
)

(define_predicate "const_1_operand"
  (and (match_code "const_int,const_wide_int,const_double,const_vector")
       (match_test "op == CONST1_RTX (GET_MODE (op))")))

(define_predicate "reg_or_1_operand"
  (ior (match_operand 0 "const_1_operand")
       (match_operand 0 "register_operand")))

;; This is used for indexing into vectors, and hence only accepts const_int.
(define_predicate "const_0_or_1_operand"
  (and (match_code "const_int")
       (ior (match_test "op == CONST0_RTX (GET_MODE (op))")
            (match_test "op == CONST1_RTX (GET_MODE (op))"))))

;; Only use branch-on-bit sequences when the mask is not an ANDI immediate.
(define_predicate "branch_on_bit_operand"
  (and (match_code "const_int")
       (match_test "INTVAL (op) >= IMM_BITS - 1")))


(define_special_predicate "vec_comparison_operator" (match_code "eq,ne,le,lt,gt,ge,leu,ltu,gtu,geu"))

(define_predicate "reg_or_sci_const"
  (ior (match_operand 0 "register_operand")
       (match_code "const_vector"))
{
        if (GET_CODE (op) == CONST_VECTOR) {
                return riscv_replicated_const_vector(op, -32, 31);
        } else return true;
}
)

(define_predicate "reg_or_sciu_const"
  (ior (match_operand 0 "register_operand")
       (match_code "const_vector"))
{
        if (GET_CODE (op) == CONST_VECTOR) {
                return riscv_replicated_const_vector(op, 0, 63);
        } else return true;
}
)


;; A legitimate CONST_INT operand that takes more than one instruction
;; to load.
(define_predicate "splittable_const_int_operand"
  (match_code "const_int")
{
  /* Don't handle multi-word moves this way; we don't want to introduce
     the individual word-mode moves until after reload.  */
  if (GET_MODE_SIZE (mode) > UNITS_PER_WORD)
    return false;

  /* Otherwise check whether the constant can be loaded in a single
     instruction.  */
  return !LUI_OPERAND (INTVAL (op)) && !SMALL_OPERAND (INTVAL (op));
})

(define_predicate "nonimmediate_operand_exclude_post"
  (match_operand 0 "nonimmediate_operand")
{
   return (!riscv_filter_pulp_operand(op, !(Pulp_Cpu>=PULP_V0)));
}
)

(define_predicate "move_operand"
  (match_operand 0 "general_operand")
{
  enum riscv_symbol_type symbol_type;

  /* The thinking here is as follows:

     (1) The move expanders should split complex load sequences into
	 individual instructions.  Those individual instructions can
	 then be optimized by all rtl passes.

     (2) The target of pre-reload load sequences should not be used
	 to store temporary results.  If the target register is only
	 assigned one value, reload can rematerialize that value
	 on demand, rather than spill it to the stack.

     (3) If we allowed pre-reload passes like combine and cse to recreate
	 complex load sequences, we would want to be able to split the
	 sequences before reload as well, so that the pre-reload scheduler
	 can see the individual instructions.  This falls foul of (2);
	 the splitter would be forced to reuse the target register for
	 intermediate results.

     (4) We want to define complex load splitters for combine.  These
	 splitters can request a temporary scratch register, which avoids
	 the problem in (2).  They allow things like:

	      (set (reg T1) (high SYM))
	      (set (reg T2) (low (reg T1) SYM))
	      (set (reg X) (plus (reg T2) (const_int OFFSET)))

	 to be combined into:

	      (set (reg T3) (high SYM+OFFSET))
	      (set (reg X) (lo_sum (reg T3) SYM+OFFSET))

	 if T2 is only used this once.  */
  switch (GET_CODE (op))
    {
    case CONST_INT:
      return !splittable_const_int_operand (op, mode);

    case CONST:
    case SYMBOL_REF:
    case LABEL_REF:
      return riscv_symbolic_constant_p (op, &symbol_type)
	      && !riscv_split_symbol_type (symbol_type);

    case HIGH:
      op = XEXP (op, 0);
      return riscv_symbolic_constant_p (op, &symbol_type)
	      && riscv_split_symbol_type (symbol_type)
	      && symbol_type != SYMBOL_PCREL;

    default:
      return true;
    }
})

(define_predicate "symbolic_operand"
  (match_code "const,symbol_ref,label_ref")
{
  enum riscv_symbol_type type;
  return riscv_symbolic_constant_p (op, &type);
})

(define_predicate "absolute_symbolic_operand"
  (match_code "const,symbol_ref,label_ref")
{
  enum riscv_symbol_type type;
  return (riscv_symbolic_constant_p (op, &type)
	  && (type == SYMBOL_ABSOLUTE || type == SYMBOL_PCREL));
})

(define_predicate "plt_symbolic_operand"
  (match_code "const,symbol_ref,label_ref")
{
  enum riscv_symbol_type type;
  return (riscv_symbolic_constant_p (op, &type)
	  && type == SYMBOL_GOT_DISP && !SYMBOL_REF_WEAK (op) && TARGET_PLT);
})

(define_predicate "call_insn_operand"
  (ior (match_operand 0 "absolute_symbolic_operand")
       (match_operand 0 "plt_symbolic_operand")
       (match_operand 0 "register_operand")))

(define_predicate "modular_operator"
  (match_code "plus,minus,mult,ashift"))

(define_predicate "equality_operator"
  (match_code "eq,ne"))

(define_predicate "order_operator"
  (match_code "eq,ne,lt,ltu,le,leu,ge,geu,gt,gtu"))

(define_predicate "signed_order_operator"
  (match_code "eq,ne,lt,le,ge,gt"))

(define_predicate "signed_order_operator_wo_eq_ne"
  (match_code "lt,le,ge,gt"))

(define_predicate "order_operator_eq_ne"
  (match_code "eq,ne"))

(define_predicate "order_operator_wo_eq_ne"
  (match_code "lt,ltu,le,leu,ge,geu,gt,gtu"))

(define_predicate "fp_native_comparison"
  (match_code "eq,lt,le,gt,ge"))

(define_predicate "fp_scc_comparison"
  (match_code "unordered,ordered,unlt,unge,unle,ungt,ltgt,ne,eq,lt,le,gt,ge"))

(define_predicate "fp_branch_comparison"
  (match_code "unordered,ordered,unlt,unge,unle,ungt,uneq,ltgt,ne,eq,lt,le,gt,ge"))

(define_predicate "tiny_memory_operand"
  (match_code "mem")
{
        rtx addr;

        /* Eliminate non-memory operations.  */
        if (GET_CODE (op) != MEM) return 0;

        if (mode == VOIDmode) mode = GET_MODE (op);
        if (GET_MODE_SIZE (mode)  > UNITS_PER_WORD) return 0;

        addr = XEXP(op, 0);
        switch (GET_CODE(addr)) {
                case SYMBOL_REF:
                        return (riscv_is_tiny_symbol_p(addr));
                case CONST:
                        return (GET_CODE (XEXP (addr, 0)) == PLUS && GET_CODE (XEXP (XEXP (addr, 0), 0)) == SYMBOL_REF &&
                                CONST_INT_P (XEXP (XEXP (addr, 0), 1)) && riscv_is_tiny_symbol_p(XEXP (XEXP (addr, 0), 0)));
                case PLUS:
                        return (GET_CODE (XEXP (addr, 0)) == REG && GET_CODE (XEXP (addr, 1)) == SYMBOL_REF &&
                                riscv_is_tiny_symbol_p(XEXP (addr, 1)));
                default: return 0;
        }
}
)

(define_predicate "tiny_ref_operand"
  (match_code "symbol_ref")
{
        return (riscv_is_tiny_symbol_p(op));
}
)

(define_predicate "tiny_ref_operand_expr"
  (match_code "const")
{
        return (GET_CODE (XEXP (op, 0)) == PLUS && GET_CODE (XEXP (XEXP (op, 0), 0)) == SYMBOL_REF &&
                                CONST_INT_P (XEXP (XEXP (op, 0), 1)) && riscv_is_tiny_symbol_p(XEXP (XEXP (op, 0), 0)));
}
)

