훅 함수이다.
```c
/* Returns one of the generic firewall policies, like NF_ACCEPT. */

unsigned int
ipt_do_table(void *priv,
         struct sk_buff *skb,
         const struct nf_hook_state *state)
{
    const struct xt_table *table = priv;
    unsigned int hook = state->hook;
    static const char nulldevname[IFNAMSIZ] __attribute__((aligned(sizeof(long))));
    const struct iphdr *ip;
    /* Initializing verdict to NF_DROP keeps gcc happy. */
    unsigned int verdict = NF_DROP;
    const char *indev, *outdev;
    const void *table_base;
    struct ipt_entry *e, **jumpstack;
    unsigned int stackidx, cpu;
    const struct xt_table_info *private;
    struct xt_action_param acpar;
    unsigned int addend;
  
    /* Initialization */
    stackidx = 0;
    ip = ip_hdr(skb);
    indev = state->in ? state->in->name : nulldevname;
    outdev = state->out ? state->out->name : nulldevname;
    /* We handle fragments by dealing with the first fragment as
     * if it was a normal packet.  All other fragments are treated
     * normally, except that they will NEVER match rules that ask
     * things we don't know, ie. tcp syn flag or ports).  If the
     * rule is also a fragment-specific rule, non-fragments won't
     * match it. */
    acpar.fragoff = ntohs(ip->frag_off) & IP_OFFSET;
    acpar.thoff   = ip_hdrlen(skb);
    acpar.hotdrop = false;
    acpar.state   = state;
  
    WARN_ON(!(table->valid_hooks & (1 << hook)));
    local_bh_disable();
    addend = xt_write_recseq_begin();
    private = READ_ONCE(table->private); /* Address dependency. */
    cpu        = smp_processor_id();
    table_base = private->entries;
    jumpstack  = (struct ipt_entry **)private->jumpstack[cpu];
  
    /* Switch to alternate jumpstack if we're being invoked via TEE.
     * TEE issues XT_CONTINUE verdict on original skb so we must not
     * clobber the jumpstack.
     *
     * For recursion via REJECT or SYNPROXY the stack will be clobbered
     * but it is no problem since absolute verdict is issued by these.
     */
    if (static_key_false(&xt_tee_enabled))
        jumpstack += private->stacksize * __this_cpu_read(nf_skb_duplicated);
  
    e = get_entry(table_base, private->hook_entry[hook]);
  
    do {
        const struct xt_entry_target *t;
        const struct xt_entry_match *ematch;
        struct xt_counters *counter;
  
        WARN_ON(!e);

        if (!ip_packet_match(ip, indev, outdev,
            &e->ip, acpar.fragoff)) {
 no_match:
            e = ipt_next_entry(e);
            continue;
        }
  
        xt_ematch_foreach(ematch, e) {
            acpar.match     = ematch->u.kernel.match;
            acpar.matchinfo = ematch->data;
            if (!acpar.match->match(skb, &acpar))
                goto no_match;
        }
  
        counter = xt_get_this_cpu_counter(&e->counters);
        ADD_COUNTER(*counter, skb->len, 1);
  
        t = ipt_get_target_c(e);
        WARN_ON(!t->u.kernel.target);
  
#if IS_ENABLED(CONFIG_NETFILTER_XT_TARGET_TRACE)
        /* The packet is traced: log it */
        if (unlikely(skb->nf_trace))
            trace_packet(state->net, skb, hook, state->in,
                     state->out, table->name, private, e);
#endif
        /* Standard target? */
        if (!t->u.kernel.targt->target) {
            int v;
  
            v = ((struct xt_standard_target *)t)->verdict;
            if (v < 0) {
                /* Pop from stack? */
                if (v != XT_RETURN) {
                    verdict = (unsigned int)(-v) - 1;
                    break;
                }
                if (stackidx == 0) {
                    e = get_entry(table_base,
                        private->underflow[hook]);
                } else {
                    e = jumpstack[--stackidx];
                    e = ipt_next_entry(e);
                }
                continue;
            }
            if (table_base + v != ipt_next_entry(e) &&
                !(e->ip.flags & IPT_F_GOTO)) {
                if (unlikely(stackidx >= private->stacksize)) {
                    verdict = NF_DROP;
                    break;
                }
                jumpstack[stackidx++] = e;
            }
  
            e = get_entry(table_base, v);
            continue;
        }
  
        acpar.target   = t->u.kernel.target;
        acpar.targinfo = t->data;
  
        verdict = t->u.kernel.target->target(skb, &acpar);
        if (verdict == XT_CONTINUE) {
            /* Target might have changed stuff. */
            ip = ip_hdr(skb);
            e = ipt_next_entry(e);
        } else {
            /* Verdict */
            break;
        }
    } while (!acpar.hotdrop);
  
    xt_write_recseq_end(addend);
    local_bh_enable();
  
    if (acpar.hotdrop)
        return NF_DROP;
    else return verdict;
}
```